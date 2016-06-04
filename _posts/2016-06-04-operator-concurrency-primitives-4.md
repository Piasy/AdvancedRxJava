---
layout: post
title: Operator 并发原语： producers（二）
tags:
    - Operator
---

原文 [Operator concurrency primitives: producers (part 2)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_86.html){:target="_blank"}

## 介绍
在[第一部分](/2016/05/18/operator-concurrency-primitives-3/){:target="_blank"}中我花了相当长的篇幅介绍了一个很复杂的 `RangeProducer`，在第二部分中，我将介绍几种更简单的 `Producer`。

你可能会思考，为什么不从更简单的 Producer 开始呢？有两个主要原因：1. 我认为介绍 `RangeProducer` 更有助于我们洞悉 Producer 的原理；2. 这些简单地 Producer 可以利用 `RangeProducer` 的思想扩展出来。

## single-producer
你对 RxJava 的 `just(T value)` 操作符肯定很熟悉了，现在我要告诉你一个秘密：它并没有实现 backpressure 和取消订阅。一旦有人订阅了它，它就会无条件执行一次 `onNext` 和 `onCompleted`：

~~~ java
Observable<Integer> source = Observable.just(1);
 
TestSubscriber<Integer> ts = new TestSubscriber<>();
ts.requestMore(0);                                     // (1)
 
source.unsafeSubscribe(ts);
 
ts.getOnNextEvents().forEach(System.out::println);
 
System.out.println("--");
 
ts.unsubscribe();                                      // (2)
 
source.unsafeSubscribe(ts);
 
ts.getOnNextEvents().forEach(System.out::println);
~~~

尽管我们在（1）处请求了 0 个数据，但我们依然会打印出一个 `1`，而且尽管我们在（2）处取消订阅了，但我们之后仍然会收到数据（第二次 `TestSubscriber` 将会打印两个 `1`）。

`just()` 的实现有问题？一定程度上可以这么说。但也不完全是这样，因为我们可以认为 `just()` 已经尽力处理取消订阅了（_虽然所谓尽力其实是什么也没做，但这依然符合 RxJava 的规范_）。

但是从 backpressure 的角度来看，它确实存在问题，只不过在现在的 RxJava 实现中，没有任何的 bug 可以体现出这个问题（例如 `MissingBackpressureException`），因为所有有限缓冲区的操作符都能毫不费力的容纳下它发出的这唯一一个数据，即便没有进行请求。由于 `just` 操作符做的如此之少，它在单一数据的 benchmark 和使用场景下性能异常突出。

RxJava 的下一个大版本（2.0），将原生兼容 [reactive-streams-jvm](https://github.com/reactive-streams/reactive-streams-jvm){:target="_blank"}，因此将会严格禁止不支持 backpressure。当然你可以利用 `onBackpressureBuffer()` 避免这一问题，但这就属矫枉过正了，我们完全可以重新实现一个更合适于 `just()` 操作符的 `Producer`。

我将一如既往的一步步加以实现，首先让我们看一下包含这一逻辑的类：

~~~ java
public final class SingleProducer<T> 
extends AtomicBoolean implements Producer {             // (1)
    final Subscriber<? super T> child;                  // (2)
    final T value;
    public SingleProducer(
            Subscriber<? super T> child, T value) {
        this.child = child;
        this.value = value;
    }
    @Override
    public void request(long n) {
        // logic comes here
    }
}
 
Integer value = 1;
Observable<Integer> just = Observable.create(child -> {
    child.setProducer(
        new SingleProducer<>(child, value));            // (3)
}); 
~~~

`SingleProducer` 继承自 `AtomicBoolean`（1），这样可以节约一次内存分配，并且持有了 child `Subscriber` 的引用，这样就可以向下游发出数据了。这样一来我们实现 `just` 就变得很简单了：我们只需要通过 `setProducer` 把 `SingleProducer` 实例设置给 child 即可。现在我们实现 `request()` 方法：

~~~ java
@Override
public void request(long n) {
    if (n < 0) {
        throw new IllegalArgumentException();    // (1)
    }
    if (n > 0 && compareAndSet(false, true)) {   // (2)
        if (!child.isUnsubscribed()) {
            child.onNext(value);                 // (3)
        }
        if (!child.isUnsubscribed()) {
            child.onCompleted();                 // (4)
        }
    }
}
~~~ 

`request()` 方法也非常简单，因为所有的请求处理都可以简化为一次 由 `false` 到 `true` 的 CAS 操作：

1. 如果请求数量是负数，就抛出 `IllegalArgumentException`。
2. 如果请求数大于零，而且 CAS 操作成功，那我们就可以进入“漏循环”了（_只会循环一次的循环_）。下游请求的具体数量无关紧要，`SingleProducer` 只会发出一个数据。
3. 如果没有取消订阅，我们就发出数据。
4. 如果没有取消订阅，我们就发出 `onCompleted()` 事件。

如果没有其他需求或者使用场景将其变得更加复杂，那 `SingleProducer` 还称不上 Advanced RxJava。

## single-delayed-producer
如果需要发出的唯一一个数据，在订阅的时候是未知的，数据将在订阅之后一段时间到达（通常是通过某些异步的处理），怎么实现？当然，忽略 backpressure 或者依赖于 `onBackpressureBuffer` 可以实现，但我们将通过更高效的基本方式解决这个问题。

为了解决这个问题，我们需要考虑一下数据到达与下游请求数据时几种原子状态的转换。我们有以下 4 种状态：

1. 没有请求，也没有数据，记为 `NO_REQUEST_NO_VALUE = 0`
2. 没有请求，但数据已经到达，记为 `NO_REQUEST_HAS_VALUE = 1`
3. 已有合法请求，但数据尚未到达，记为 `HAS_REQUEST_NO_VALUE = 2`
4. 已有合法请求，且此时数据已经到达，记为 `HAS_REQUEST_HAS_VALUE = 3`

我们通过继承 `AtomicInteger` 来记录一个状态变量，并且在上面的条件满足时通过 CAS 进行状态转换。

首先我们看一下包含这一逻辑的类以及简单的使用场景：

~~~ java
public class SingleDelayedProducer<T> 
extends AtomicInteger implements Producer {
    private static final long serialVersionUID = 1L;
    final Subscriber<? super T> child;
    T value;                                                        // (1)                     
    static final int NO_REQUEST_NO_VALUE = 0;
    static final int NO_REQUEST_HAS_VALUE = 1;
    static final int HAS_REQUEST_NO_VALUE = 2;
    static final int HAS_REQUEST_HAS_VALUE = 3;
  
    public SingleDelayedProducer(
            Subscriber<? super T> child) {
        this.child = child;
    }
     
    @Override
    public void request(long n) {
        // implement request
    }
     
    public void set(T value) {
        // implement set
    }
}
 
Observable<Integer> justDelayed = Observable.create(child -> {
    SingleDelayedProducer<Integer> p = 
        new SingleDelayedProducer<>(child);
    ForkJoinPool.commonPool().submit(() -> {
        try {
            Thread.sleep(500);                                     // (2)
        } catch (InterruptedException ex) {
            child.onError(ex);
            return;
        }
        p.set(1);                                                  // (3)
    });
    child.setProducer(p);
});
 
justDelayed.subscribe(System.out::println);
 
Thread.sleep(1000);
~~~

在 `SingleDelayedProducer` 中，我们不能把 `value` 声明为 `final`，因为在构造之后我们需要为其赋值（1）。在构造 Observable 时，我们在订阅时启动一个后台任务，它会在把数据设置给 Producer（3）之前 sleep 500 毫秒（2）。

在 `request()` 的实现中，我们尝试从 `NO_REQUEST_NO_VALUE` 切换到 `HAS_REQUEST_NO_VALUE` 并退出，或者从 `NO_REQUEST_HAS_VALUE` 切换到 `HAS_REQUEST_HAS_VALUE` 并发出数据：

~~~ java
// ...
@Override
public void request(long n) {
    if (n < 0) {
        throw new IllegalArgumentException();        // (1)
    }
    if (n == 0) {
        return;
    }
    for (;;) {                                       // (2)
        int s = get();
        if (s == NO_REQUEST_NO_VALUE) {              // (3)
            if (!compareAndSet(
                    NO_REQUEST_NO_VALUE, 
                    HAS_REQUEST_NO_VALUE)) {
                continue;                            // (4)
            }
        } else if (s == NO_REQUEST_HAS_VALUE) {      // (5)
            if (compareAndSet(
                    NO_REQUEST_HAS_VALUE, 
                    HAS_REQUEST_HAS_VALUE)) {
                if (!child.isUnsubscribed()) {       // (6)
                    child.onNext(value);
                }
                if (!child.isUnsubscribed()) {
                    child.onCompleted();
                }
            }                                        // (7)
        }
        return;                                      // (8)
    }
}
// ...
~~~

代码看起来很复杂，但别怕，只不过是常量名字有点长：

1. 和普通 Producer 一样，我们对请求数量进行检查。
2. 我们需要一个 for 循环，因为对 `request()` 和 `set()` 的调用都会改变状态，所以我们需要重试状态转换，或者是返回。
3. 如果当前处于 `NO_REQUEST_NO_VALUE` 状态，我们就尝试切换到 `HAS_REQUEST_NO_VALUE`，如果成功，我们就可以返回了，后续调用 `set()` 时将会发出数据。
4. 如果切换失败，说明有并发的调用改变了状态（_此时不是 `NO_REQUEST_NO_VALUE` 状态了_），所以我们需要重新循环，并重新进行判断。
5. 如果在（_当前_）请求到达之前，数据已经到达（`NO_REQUEST_HAS_VALUE`），我们就尝试切换到 `HAS_REQUEST_HAS_VALUE`。
6. 如果成功，我们就有条件地发出数据以及 `onCompleted` 事件，然后返回。CAS 操作也将保证后续对 `request()` 或者 `set()` 的调用不会执行任何操作（_译者注：CAS 会保证内存同步，这样后续并发的调用都能看到最新的 `HAS_REQUEST_HAS_VALUE` 状态，根据循环的逻辑，此状态下我们不会进行任何操作_）。
7. 如果失败，那就说明状态被并发调用改变了：有并发的线程成功进行了这个 CAS，或者执行了其他的状态切换。这时我们都可以安全返回（因为状态不会被逆向切换）。
8. 如果当前处于其他的状态，我们直接返回。

`set()` 方法比较类似，但是执行了不同的状态检查和状态切换：

~~~ java
    // ...
    public void set(T value) {
        for (;;) {                                       // (1)
            int s = get();
            if (s == NO_REQUEST_NO_VALUE) {
                this.value = value;                      // (2)
                if (!compareAndSet(                      // (3)
                        NO_REQUEST_NO_VALUE, 
                        NO_REQUEST_HAS_VALUE)) {
                    continue;
                }
            } else if (s == HAS_REQUEST_NO_VALUE) {      // (4)
                if (compareAndSet(
                        HAS_REQUEST_NO_VALUE, 
                        HAS_REQUEST_HAS_VALUE)) {
                    if (!child.isUnsubscribed()) {
                        child.onNext(value);
                    }
                    if (!child.isUnsubscribed()) {
                        child.onCompleted();
                    }
                }
            }
            return;                                      // (5)
        }
    }
}
~~~

1. 由于状态可能会被并发调用改变，所以我们需要一个循环。
2. 如果我们处于 `NO_REQUEST_NO_VALUE` 状态，我们就先设置 `value`。如果有 `set()` 方法被并发调用，那这里就存在竞争问题。但是在我们的例子中，我们确信只会调用一次 `set()`，因此这里不存在问题。如果你实现了一个 publisher，而且 `set()` 方法允许并发调用，那你就需要通过某种方式避免竞争问题（最简单的方法就是为 `value` 加上 `volatile` 关键字，让缓存一致性来决定最终哪个值将被设置成功）。设置完之后，我们就尝试切换到 `NO_REQUEST_HAS_VALUE`。
3. 如果切换成功，我们就可以返回了，发射数据就是 `request()` 的责任了。如果切换失败，我们就需要重新循环并检查状态。
4. 如果我们处在 `HAS_REQUEST_NO_VALUE` 状态，那我们就可以尝试切换到 `HAS_REQUEST_HAS_VALUE` 状态，如果切换成功，我们就发出数据和 `onCompleted` 事件。如果切换失败，说明并发线程已经发出了数据，我们就可以返回了。
5. 如果当前处于其他的状态，我们直接返回。

## 总结
在这篇文章中，我介绍和实现了两种只发射单一数据、处理了 backpressure 和取消订阅的 Producer。尽管直接发出数据的版本（`SingleProducer`）在实际情况下使用较少，但是 `SingleDelayedProducer` 却有很多使用场景，尤其是有人想在 `onError()` 中发出一个数据（例如 `onErrorReturn()`），或者想在 `onCompleted()` 中发出数据（例如 `toList()` 和 `buffer()`），这些场景下通常都是只发出一个数据之后结束（_数据也通常在订阅之后才到达_）。

在下一篇文章中，我将扩展 delayed producer 的概念（就像 `from()` 那样直接发出多个数据），并让它不仅仅只能发出单一数据，还能**在下游请求时发出多个数据**。
