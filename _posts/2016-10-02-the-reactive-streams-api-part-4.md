---
layout: post
title: Reactive-Streams API（四，完结）：SubscriptionArbiter 的使用
tags:
    - Reactive Stream
---

原文 [The Reactive-Streams API (part 4 - final)](http://akarnokd.blogspot.com/2015/06/the-reactive-streams-api-part-4-final.html){:target="_blank"}

## 介绍

在这篇介绍 Reactive-Streams API 的最后一篇文章中，我会讲讲我们对 `SubscriptionArbiter`（`ProducerArbiter` 的一个兄弟） 的高频使用需求，这一点可能会让很多人感到惊讶，当涉及到多个数据源、调度、其他异步内容时，我们将会很频繁的用到它。

在 RxJava 1.x 中，给 `Subscriber` 设置 `Producer` 是可选的，我们可以在没有 `Producer` 的情况下直接调用 `onError()` 和 `onCompleted()`。这些调用最终都会调用 `rx.Subscriber.unsubscribe()`，并清理相关资源。

与之相反，RS 要求 `Subscription` 必须在 `onXXX` 被调用之前，通过 `onSubscribe()` 传递给 `Subscriber`。

在本文中，我将展示一些这一要求可能导致的困境，尤其是当操作符的实现方式是典型的 RxJava 结构时。

## 稍后订阅

`defer()` 是必须考虑“订阅之前的错误”（error-before-Subscription）情况的操作符之一。当订阅一个推迟的 `Publisher` 时，操作符会先利用用户提供的工厂方法创建一个新的 `Publisher`，这个新的才是之后被订阅的。由于我们必须对用户的代码进行检查，所以我们就要捕获可能的异常，并且把异常通知给下游。但是，在这种情况下（要调用 `child.onError()`），我们就需要 child Subscriber 已经有 Subscription 了，但如果 child 这时已经有了 Subscription，那新生成的 Producer 收到 Subscription 时就不能转交给 child 了。解决方案就是使用一个 `SubscriptionArbiter`，它让我们可以提前发送错误事件，或者稍后切换到“真正的”数据源。

（_译者注：这里 SubscriptionArbiter 相当于一个占位符或者说管道，Subscriber 不能替换 Subscription，但 SubscriptionArbiter 可以，这种移花接木的思路很赞，像国内很多插件化/热修复的方案，使用自定义的 ClassLoader，想法上差不多_）

~~~ java
public final class OnSubscribeDefer<T>
implements OnSubscribe<T> {
    final Func0<? extends Publisher<? extends T>> factory;
    public OnSubscribeDefer(
           Func0<? extends Publisher<? extends T>> factory) {
        this.factory = factory;
    }
    @Override
    public void call(Subscriber<? super T> child) {
         
        SubscriptionArbiter sa = new SubscriptionArbiter();
        child.onSubscribe(sa);                                 // (1)  
         
        Publisher<? extends T> p;
        try {
            p = factory.call();
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            child.onError(e);                                  // (2)
            return;
        }
        p.subscribe(new Subscriber<T>() {                      // (3)
            @Override
            public void onSubscribe(Subscription s) {
                sa.setSubscription(s);                         // (4)
            }
 
            @Override
            public void onNext(T t) {
                child.onNext(t);
            }
 
            @Override
            public void onError(Throwable t) {
                child.onError(t);
            }
 
            @Override
            public void onComplete() {
                child.onComplete();
            }
             
        });
    }
}
~~~

这一次我们不需要管理资源，但我们需要处理 `Subscription` 切换：

1. 首先我们创建一个空的 arbiter，把它设置给 child。
2. 如果用户的工厂方法抛出了异常，我们就可以安全地给 child 发送 onError 了，因为它已经有了 Subscription（尽管是个 arbiter）。
3. 我们不能直接订阅到 child（因为它已经有了 Subscription），所以我们创建一个新的 Subscriber，并重写 `onSubscribe` 方法。
4. 我们把“真正的” Subscription 设置给 arbiter，其他的事件直接转发给 child 即可。

## 延迟一段时间后订阅

让我们看看现在的 `delaySubscription()` 操作符的实现，它把实际订阅操作延迟了一定的时间。我们几乎可以把已有的实现代码拷贝过来，但由于 API 发生了变化，会发生编译问题：

~~~ java
public final class OnSubscribeDelayTimed<T>
implements OnSubscribe<T> {
    final Publisher<T> source;
    final Scheduler scheduler;
    final long delay;
    final TimeUnit unit;
    public OnSubscribeDelayTimed(
            Publisher<T> source, 
            long delay, TimeUnit unit, 
            Scheduler scheduler) {
        this.source = source;
        this.delay = delay;
        this.unit = unit;
        this.scheduler = scheduler;
    }
    @Override
    public void call(Subscriber child) {
        Scheduler.Worker w = scheduler.createWorker();
         
        // child.add(w);
         
        w.schedule(() -> {
            // if (!child.isUnsubscribed()) {
 
                source.subscribe(child);
 
            // }
        }, delay, unit);
    }
}
~~~

我们不能直接把资源添加到 child 中，也不能检查它是否已经被取消订阅（RS 中没有这两个方法了）。为了能清除 worker，我们就需要一个 disposable 容器，为了能够取消订阅，我们还需要一个可以“重放”数据源提供的 `Subscription` 取消操作的东西：

~~~ java
@Override
public void call(Subscriber<? super T> child) {
    Scheduler.Worker w = scheduler.createWorker();
     
    SubscriptionArbiter sa = new SubscriptionArbiter();    // (1)
 
    DisposableSubscription dsub = 
        new DisposableSubscription(
            sa, new DisposableList());                     // (2)
     
    dsub.add(w);                                           // (3)
     
    child.onSubscribe(dsub);                               // (4)
     
    w.schedule(() -> {
        source.subscribe(new Subscriber<T>() {             // (5)
            @Override
            public void onSubscribe(Subscription s) {
                sa.setSubscription(s);                     // (6)
            }
 
            @Override
            public void onNext(T t) {
                child.onNext(t);
            }
 
            @Override
            public void onError(Throwable t) {
                child.onError(t);
            }
 
            @Override
            public void onComplete() {
                child.onComplete();
            }
             
        });
    }, delay, unit);
~~~

比原来的实现多了很多代码，但值得这样做：

1. 我们需要一个 `SubscriptionArbiter` 来进行占位，因为实际的 `Subscription` 会延迟到来，所以我们需要用 arbiter 先记录下取消操作。
2. 如果 child 要取消整个操作，那我们就需要取消这一次调度（直接取消这个 worker）。但由于 arbiter 没有资源管理能力，所以我们需要一个 disposable 容器。当然，我们只有一个资源，用不着一个 List，所以你可以实现一个自己的单一 disposable 容器类。
3. 我们把 worker 加入到容器中，它就会替我们处理取消的事宜了。
4. 当我们设置好 arbiter 和 disposable 之后，我们就可以把它们交给 child 了。然后 child 就可以随意进行 `request()` 以及 `cancel()` 操作了，arbiter 和 disposable 会在合适的时机（_实际 Subscription 到来时_），把记录下来的操作都转发给数据源/资源了。
5. 由于我们已经给 child 设置过了 Subscription，所以我们不能在调度时直接使用 child。我们创建一个 wrapper，在收到 Subscription 时设置给 arbiter，并且转发其他的 onXXX 事件。
6. 我们在把实际的 Subscription 设置给 arbiter 时，arbiter 会重放积累的 request/cancel 操作。

现在你可能觉得最终调度时的 `Subscriber`（5）有错误，它没有在 `onNext` 中调用 `sa.produced(1)`。这确实会导致请求量计数不一致，但是一旦实际的 `Subscription` 被设置之后，后续 child 的 `request(n)` 都会原封不动地转发给上游，而我们后面又不会再调用 `setSubscription()` 了。所以上游能收到正确的请求量，即便 arbiter 计数不一致，也不会导致任何问题。为了保证更安全，你可以：

+ 在 `onNext` 中调用 `sa.produced(1)`；
+ 或者实现一个自己的 arbiter，只接受一个 `Subscription`，并且在收到它之后停止计数。

## 总结

在本文中，我展示了两种使用 `SubscriptionArbiter` 的场景。幸运的是，不是所有的操作符都需要进行这样的操作，但主流的都需要，因此在处理“实际” `Subscription` 延迟抵达以及 `cancel()` 的同时释放资源的问题时，我们需要两种特殊的 `Subscription`（_`SubscriptionArbiter` 和 `DisposableSubscription`_）

由于这种情况会频繁出现，我相信 RxJava 2.0 会提供一种标准而且高效的实现方案，来帮助我们遵循 Reactive-Streams 规范。

作为 RS API 系列的总结，RS 最小的接口集合就能满足典型异步数据流场景的需求了。我们可以把 RxJava 的思想迁移到 RS 中，但基础设施以及一些辅助工具都需要从头开始实现。我已经展示了几种基本的 `Subscription`：`SingeSubscription`，`SingleDelayedSubscription`，`SubscriptionArbiter` 以及 `DisposableSubscription`。再加上一些其他类似的工具类，它们将是实现 RxJava 2.0 操作符的主要工具类。

在下一个系列中，我将讲解响应式编程中争议最多的一个类型（_`Subject`_），而且我不仅会讲到它们的细节，还会讲如何实现我们自己的变体。
