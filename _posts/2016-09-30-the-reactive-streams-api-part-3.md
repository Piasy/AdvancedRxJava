---
layout: post
title: Reactive-Streams API（三）：资源管理与 TakeUntil
tags:
    - Reactive Stream
---

原文 [The Reactive-Streams API (part 3)](http://akarnokd.blogspot.com/2015/06/the-reactive-streams-api-part-3.html){:target="_blank"}

## 介绍

在本文中，我将讲解如何把 `rx.Subscriber` 管理资源的能力移植到 Reactive-Streams API 中。但是由于 RS 并未明确任何资源管理方面的要求，所以我们需要引入（把 `rx.Subscription` 重命名）我们自己的容器类型，并把它加入到 RS 的 `Subscriber` 的取消逻辑中。

## Subscription vs. Subscription

为了避免造成困惑，RxJava 2.0 把  `XXXSubscription` 替换为了 `XXXDisposable`，我不会在这里详细介绍这些类，但是会讲几个资源管理的基本接口：

~~~ java
interface Disposable {
    boolean isDisposed();
    void dispose();
}
 
interface DisposableCollection extends Disposable {
    boolean add(Disposable resource);
    boolean remove(Disposable resource);
    boolean removeSilently(Disposable resource);
 
    void clear();
    boolean hasDisposables();
    boolean contains(Disposable resource);
}
~~~

使用的规则是一样的：线程安全、幂等性。

## `DisposableSubscription`

加入资源管理最基本的方式就是对 `Subscription` 进行一次包装，拦截 `cancel()` 调用并且调用底层容器类的 `dispose()`：

~~~ java
public final class DisposableSubscription
implements Disposable, Subscription {
    final Subscription actual;
    final DisposableCollection collection;
    public DisposableSubscription(
            Subscription actual, 
            DisposableCollection collection) {
        this.actual = Objects.requireNonNull(actual);
        this.collection = Objects.requireNonNull(collection);
    }
    public boolean add(Disposable resource) {
        return collection.add(resource);
    }
    public boolean remove(Disposable resource) {
        return collection.remove(resource);
    }
    public boolean removeSilently(Disposable resource) {
        return collection.remove(resource);
    }
    @Override
    public boolean isDisposed() {
        return collection.isDisposed();
    }
    @Override
    public void dispose() {
        cancel();
    }
     
    @Override
    public void cancel() {
        collection.dispose();
        actual.cancel();
    }
    @Override
    public void request(long n) {
        actual.request(n);
    }
}
~~~

由于 `DisposableSubscription` 也实现了 `Disposable` 接口，所以它也可以被加入到容器中，构成一个复杂的 dispose 网络。但是绝大多数情况下，我们都希望避免额外的内存分配，因此，上面的这些代码可能会被融入到其他的类型中，例如在 `lift()` 调用中创建的 `Subscriber` 类型。

如果你熟悉 RxJava 的规范，以及[操作符实现的陷阱之一](/AdvancedRxJava/2016/05/29/pitfalls-of-operator-implementations/#unsubscribing-the-downstream){:target="_blank"}，那就不应该取消订阅下游，因为这可能会导致资源的提前释放。

（_这也是目前 RxAndroid 的 `LifecycleObservable` 的一个 bug，当我们在中间插入一个类似于 `takeUntil()` 的操作符时，它不会向下游发送 `onCompleted()`，而是取消订阅了下游。_）

在 RS 中，取消订阅下游实际上是不会发生的。每一层都要么只能原封不动的转发 `Subscription`（因此无法添加资源），要么只能把它包装为 `DisposableSubscription` 这样的类型，然后依然把下游当做一个 `Subscription` 进行转发。如果你在这一层调用了 `cancel()`，你是无法调用包装 Subscriber 的类的 `cancel()` 的。

当然，你非要搞破坏那肯定是可以的，但 RS 比 RxJava 做了更多的努力，而且原则是不变的：不应该 cancel/dispose 下游的资源，或者在操作符链条中共享资源。

## `TakeUntil`

现在让我们看看怎么实现能够管理外部资源的 `takeUntil()` 操作符（我把代码拆分了一下，更方便阅读）：

~~~ java
public final class OperatorTakeUntil<T> 
implements Operator<T, T> {
    final Publisher<?> other;
    public OperatorTakeUntil(Publisher<?> other) {
        this.other = Objects.requireNonNull(other);
    }
    @Override
    public Subscriber<? super T> call(
            Subscriber<? super T> child) {
        Subscriber<T> serial = 
            new SerializedSubscriber<>(child);
 
        SubscriptionArbiter arbiter = 
            new SubscriptionArbiter();                       // (1)
        serial.onSubscribe(arbiter);
         
        SerialDisposable sdUntil = new SerialDisposable();   // (2)
        SerialDisposable sdParent = new SerialDisposable();  // (3)
~~~

到目前为止，看起来都和 RxJava 的实现类似：我们把 child 包装为一个 `SerializedSubscriber`，防止 `Publisher` 并行发出 `onError()` 和 `onCompleted()`。

我们创建了一个 `SubscriptionArbiter`（一个 `ProducerArbiter` 的变体），主要是考虑以下原因：假设我们正在 `call()` 函数中，另一个已经订阅的数据源发出了一个数据，那我们就需要把数据转发到 child Subscriber 中，然而在我们得到 `Subscription` 之前（_调用 `onSubscribe()` 之前_），我们是无法在其上调用 `onXXX` 函数的，所以我们只能等到操作符链条上调用 `onSubscribe()` 之后（_拿到 `Subscription` 之后_），才可以转发数据。我会在下一篇文章中更详细地讲解这个问题。

然而，由于取消订阅的机会在 `Subscriber` 那里（_我们调用 `Subscription.cancel()` 取消订阅，而我们会调用 `Subscriber.onSubscribe(Subscription)` 把 `Subscription` 交给 `Subscriber`_），我们需要把 `Subscription` 从子 `Subscriber` 传递到父 `Subscriber` 中（2），这样它们中的任意一个到达终止状态时，它都能 `cancel()` 另一个。由于子 Subscriber 可能比父 Subscriber 先收到 Subscription 和取消事件，我们也将需要反过来取消父 Subscriber（3）。

~~~ java
// ...
Subscriber<T> parent = new Subscriber<T>() {
    DisposableSubscription dsub;
    @Override
    public void onSubscribe(Subscription s) {
        DisposableSubscription dsub = 
                new DisposableSubscription(s, 
                new DisposableList());              // (1)
        dsub.add(sdUntil);                          // (2)
        sdParent.set(dsub);
        arbiter.setSubscription(dsub);              // (3)
    }
    @Override
    public void onNext(T t) {
        serial.onNext(t);
    }
    @Override
    public void onError(Throwable t) {
        serial.onError(t);
        sdParent.cancel();                          // (4)
    }
    @Override
    public void onComplete() {
        serial.onComplete();
        sdParent.cancel();
    }
};
~~~

父 Subscriber 的实现略有不同，我们需要处理后来的 Subscription，并且建立好取消订阅的链条：

1. 我们创建了一个 `DisposableSubscription`，底层使用基于 List 的 Disposable 集合。
2. 我们把包装了 `Disposable`（指向另一个 `Subscription`） 的 `SerialDisposable` 加入到容器中。我们也把 `DisposableSubscription` 加入到 `sdParent` 中，这就让另一个 `Subscriber` 可以在 parent 开始之前结束自己。
3. 我们把包装好的对象加入到 `arbiter` 中。
4. 当错误事件发生时，我们要确保取消掉容器。而由于容器中包含了另一个 `Subscription`，所以整个事件流也会被取消。

最后我们需要为另一个事件流创建 `Subscriber`，并且确保它和 parent 连接起来：

~~~ java
        // ...
        Subscriber<Object> until = new Subscriber<Object>() {
            @Override
            public void onSubscribe(Subscription s) {
                sdUntil.set(Disposables.create(s::cancel));    // (1)
                s.request(Long.MAX_VALUE);
            }
            @Override
            public void onNext(Object t) {
                parent.onComplete();                           // (2)
            }
            @Override
            public void onError(Throwable t) {
                parent.onError(t);
            }
            @Override
            public void onComplete() {
                parent.onComplete();
            }
        };
         
        this.other.subscribe(until);
         
        return parent;
    }
}
~~~

我们从另一个数据源接收到 `Subscription` 时，我们就把它包装为一个 `Disposable`（和现在 `Subscription.create()` 的做法一样）。由于 `Disposable` 的终结状态特性，即便我们的主要事件流在另一个流接收到 `Subscription` 之前就已经结束，包装的 `SerialDisposable` 依然会被取消，而这就会立即取消刚刚接收到的 `Subscription`。（_译者注：对于“终结状态特性”，不了解的朋友可以看看之前的文章：[Operator 并发原语： subscription-containers（一）](/AdvancedRxJava/2016/07/15/operator-concurrency-primitives-subscription-containers-1/#section){:target="_blank"}_）

注意，由于取消/释放资源的能力取决于时间和顺序，所以我们通常都需要为每个资源创建一个容器（例如 `sdParent`、`sdOther`），这样无论哪个 `Subscription` 在任何时候到达时，我们都能释放所有的资源。

## `TakeUntil` v2

如果仔细看看上面 `takeUntil()` 的实现，就会发现我们对各种 `Subscription` 进行了重新组织，我们其实可以理清 `Disposable` 导致的混乱：

~~~ java
@Override
// ...
public Subscriber<? super T> call(Subscriber<? super T> child) {
    Subscriber<T> serial = new SerializedSubscriber<>(child);
 
    SubscriptionArbiter sa = new SubscriptionArbiter();        // (1)
 
    DisposableSubscription dsub = 
        new DisposableSubscription(sa, new DisposableList());  // (2)
     
    serial.onSubscribe(dsub);                                  // (3)
     
    Subscriber<T> parent = new Subscriber<T>() {
        @Override
        public void onSubscribe(Subscription s) {
            dsub.add(Disposables.create(s::cancel));           // (4)
            sa.setSubscription(s);                             // (5)
        }
        // ...
    };
     
    Subscriber<Object> until = 
    new Subscriber<Object>() {
        @Override
        public void onSubscribe(Subscription s) {
            dsub.add(Disposables.create(s::cancel));           // (6)
            s.request(Long.MAX_VALUE);
        }
        // ...
~~~

它的原理如下：

1. 我们创建了一个 `SubscriptionArbiter`。
2. 然后把它包装为一个 `DisposableSubscription`。
3. 然后把它推到下游。这种组合能保证任何的取消以及请求，都会积累到 arbiter 接收到一个“真正的” `Subscription` 时。
4. 一旦主流收到 `Subscription` 之后，我们就把它包装为一个 `Disposable` 并加入到 `dsub` 容器中。
5. 然后我们就更新 arbiter 的 `Subscription`：所有积累的请求以及取消操作都会“重放”到上游的 `Subscription` 上。
6. 当另一条流收到 `Subscription` 之后，我们也把它包装为一个 `Disposable` 并加入到 `dsub` 容器中。

最后 parent 会在它的 `onError()` 和 `onComplete()` 中调用 `dsub.dispose()` 了。

让我们梳理一下各种取消的路径：

+ 下游取消：下游会取消 `dsub`，`dsub` 会取消 arbiter，arbiter 会取消任何收到的 `Subscription`。
+ 主流结束：主流会取消 `dsub`，`dsub` 会取消 arbiter，arbiter 会取消上游的 `Subscription`。此外，`dsub` 也会取消其他存在的 `Subscription`。
+ 支流结束：支流会取消 `dsub`，`dsub` 会取消 arbiter。一旦主流收到 `Subscription` 之后，`dsub` 和 arbiter 都会立即取消这个 `Subscription`。

## 总结

在本文中，我讲解了怎么在 Reactive-Stream 的 `Subscriber` 和 `Subscription` 体系中实现资源管理，以及演示了如何实现一个 `takeUntil()` 操作符。

尽管看起来我们创建了和 RxJava 实现中同样多（甚至更多）的对象，但是很多操作符都不需要资源管理（例如 `take()`），甚至都不需要包装收到的 `Subscription` 对象（例如 `map()`）。

在下一篇（最后一篇）关于 RS API 的文章中，我将介绍我们对各种 arbiter 类型更大的需求（在本文的 `takeUntil()` 例子中已经有所涉及），因为我们必须给 `Subscriber` 设置一个 `Subscription` 并且还要保证取消功能的正常，此外即便数据源发生了变化我们也不能调用多次 `onSubscribe()`。
