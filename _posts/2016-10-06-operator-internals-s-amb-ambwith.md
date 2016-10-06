---
layout: post
title: 深入理解 Operator：Amb 和 AmbWith
tags:
    - Operator
---

原文 [Operator internals: Amb, AmbWith](http://akarnokd.blogspot.com/2015/10/operator-internals-amb-ambwith.html){:target="_blank"}

## 介绍

`amb` 是 `ambiguous` 的缩写，它的输入是一个 `Observable` 集合，输出的是转发第一个发出事件的 Observable 的所有后续事件，并且取消订阅其他所有的 Observable，也会忽略它们的任何事件。amb 支持 backpressure。

从编写操作符的角度出发，我们需要考虑以下几个属性/要求：

+ 当下游订阅的时候，数据源 `Observable` 的数量是已知的。
+ 我们需要一个 `CompositeSubscription` 之外的容器类，因为它不支持从中挑出最先发事件的然后取消其他的。
+ 我们需要把下游的请求转发给所有的数据源。
+ 在我们订阅数据源的过程中，取消订阅、请求数据、甚至是“赢家”的出现，都随时可能发生。

两个大版本间的实现方式稍有不同。

## 1.x 的实现

1.x 的实现有些冗余。它利用 `ConcurrentLinkedQueue` 来保存 `AmbSubscriber` 列表，然后胜出的 AmbSubscriber 被保存在一个 `AtomicReference` 中。

为了处理取消订阅，我们就需要在 child 中注册一个取消订阅的回调，这样在取消订阅的时候，我们就可以逐个把数据源取消订阅了。

当下游订阅的时候，我们会遍历所有的数据源，为每个 Observable 创建一个 AmbSubscriber，然后订阅它。由于取消订阅或者胜出都可能在遍历过程中发生，所以我们在遍历过程中需要检查这两种情况，以便尽早退出循环。

当所有的数据源都订阅之后，我们就会为 child 设置一个 `Producer`。它的任务就是把 child 的请求转发给所有的 AmbSubscriber，或者有数据源胜出之后，只转发给胜利者。

这看起来有点奇怪，因为如果在设置 Producer 之前就有了胜利者，那我们也就没必要设置 Producer 了，因为这个胜利者并没有按照 backpressure 的要求行事（_译者注：都还没有设置 Producer，也就不会有请求，没有收到请求就发出了数据，这就是没有遵守 backpressure 的要求嘛_）。

在 AmbSubscriber 中，收到任何事件之后，它都会检查自己是不是胜利者，如果是，就把事件转发给 child。否则，它会尝试用 CAS 操作把自己设置为胜利者，如果成功了，它会取消订阅所有其他的 AmbSubscriber（_并且转发事件_），如果失败了，那就把自己取消订阅掉。

## 2.x 的实现

2.x 的实现会简洁一些，而且还会利用“被下游订阅时数据源的数量是已知的”这一事实。所以 `AmbInnerSubscriber` 将会用一个数组保存，而胜利者标志则用一个 `volatile int` 来记录，它的更新是通过一个“成员变量更新器（field updater）”实现的（_译者注：当我翻译到这篇文章时，一年时间过去了，现在 winner 已经换成了用 `AtomicInteger` 来实现。_）。

当 child 订阅时，我们会进行两次循环，第一次我们会为每个数据源创建一个 AmbInnerSubscriber，然后在 child 中加入一个自定义的 `Subscription` 对象（`AmbCoordinator`），第二次我们会逐个订阅数据源。第二次循环过程中会检查是否已经有了胜利者。

`winner` 成员变量在不同的状态下有不同的含义。-1 表示 child 已经取消订阅了，0 表示当前没有胜利者，正数则表示胜利者的下标加一。

在响应式编程的世界里，`Subscription` 比数据请求、取消订阅来得晚的可能性在逐渐增加，所以我们必须考虑这种情况。因此 AmbInnerSubscriber 必须用一个 volatile 成员来保存它的 Subscription，而且还要用另一个成员记录积累的请求量。这种模式在 2.x 中用得非常多，所以我们在有一个专门的类负责这件事：

~~~ java
class AmbSubscriber<T> 
extends AtomicReference<Subscription>
implements Subscriber<T>, Subscription {
    volatile long missedRequested;
    static final AtomicLongFieldUpdater MISSED_REQUESTED = ...;
 
    static final Subscription CANCELLED = ...;
}
~~~

它当然实现了 `Subscriber`，为了方便，也实现了 `Subscription`（这样我们就有 `request()` 和 `cancel()` 了）。它有一个 Subscription 常量 `CANCELLED`，用来表示已被取消订阅，并且告知迟来的 `request()` 和 `cancel()` 不要做任何事了。通过继承自 `AtomicReference`，这样我们就可以利用原子性接口保存和访问将要到来的 Subscription 了（_译者注：这里违背了“组合优于继承”原则，但带来了性能提升_）。

让我们首先看看 `cancel()` 的实现：

~~~ java
@Override
public void cancel() {
    Subscription s = get();
    if (s != CANCELLED) {
        s = getAndSet(CANCELLED);
        if (s != CANCELLED && s != null) {
            s.cancel();
        }
    }
}
~~~

现在这段代码看起来应该很熟悉了。当 `cancel()` 被调用时，如果当前的 `Subscription` 不是 CANCELLED，那我们就尝试把它设置为 CANCELLED。原子操作函数确保了只会有一个线程成功地把当前的 Subscription 置为 CANCELLED，而这个线程负责将其取消。注意，`cancel()` 有可能在 `onSubscribe` 之前被调用，所以它是有可能为 `null` 的，我们要检查。

接下来，我们看看 `onSubscribe()`：

~~~ java
@Override
public void onSubscribe(Subscription s) {
    if (!compareAndSet(null, s)) {                         // (1)
        s.cancel();                                        // (2)
        if (get() != CANCELLED) {                          // (3)
            SubscriptionHelper.reportSubscriptionSet();
        }
        return;
    }
             
    long r = MISSED_REQUESTED.getAndSet(this, 0L);         // (4)
    if (r != 0L) {                                         // (5)
        s.request(r);
    }
}
~~~

1. 我们利用 CAS 操作用新来的 Subscription 替换 null。
2. 如果 CAS 失败，则说明已经有了 Subscription，那我们就把新来的取消掉。
3. 有可能有些恶意的数据源会调用多次 `onSubscribe`，尽管可能性不大，但我们还是需要处理。如果此时的 Subscription 不是 CANCELLED，我们就报告这种情况的出现，然后返回。
4. 如果 CAS 成功，我们就用 `getAndSet` 读取所有积累的请求量。
5. 如果确实有积累的请求，那我们就转发给新来的 Subscription。

最后我们看看 `request()`：

~~~ java
@Override
public void request(long n) {
    Subscription s = get();
    if (s != null) {                                       // (1)
        s.request(n);
    } else {
        BackpressureHelper.add(MISSED_REQUESTED, this, n); // (2)
        s = get();
        if (s != null && s != CANCELLED) {                 // (3)
            long r = MISSED_REQUESTED.getAndSet(this, 0L); // (4)
            if (r != 0L) {                                 // (5)
                s.request(r);
            }
        }
    }
}
~~~

1. 首先我们检查当前是否已经有了 Subscription，如果有，就直接向它请求。当前的 Subscription 有可能是 CANCELLED，但没关系，它不会做任何事情。
2. 我们利用 `BackpressureHelper` 来安全累加请求计数 `missedRequested`（它会保证不发生上溢，最多加到 `Long.MAX_VALUE`）。**_2.x 的 bug：没有对 n 进行合法性检查_**。
3. 当我们累加了请求计数之后，我们还要再次检查是否已经有了 Subscription，因为它可能被异步地设置。
4. 如果当前有了 Subscription 且不是 CANCELLED，那我们就用 `getAndSet` 读出积累的请求量。这个原子调用保证了请求计数只会被 `request()` 或者 `onSubscribe()` 之一读取到。
5. 如果请求计数不为零，那我们就向 Subscription 发出请求。否则可能 `onSubscribe()` 已经请求过了，或者压根就没有请求。

`AmbInnerSubscriber.onXXX` 和 1.x 的实现基本一样。它有一个自己的变量 `boolean won`（不需要 volatile），如果是 true，就作为事件派发的快路径。否则它就会尝试把自己设置为胜利者，如果成功，就会置 won 为 true。否则就会取消订阅当前持有的 Subscription（注意这时 Subscription 不可能为 null，因为 RS 要求在 onXXX 之前必须调用 onSubscribe）。

**_2.x 的 bug：如果 AmbSubscriber 胜利了，它并不会取消其他的 AmbSubscriber，这样它们会一直保持订阅状态_**。

## 总结

`amb` 操作符并不复杂，顶多只能算是中等水平。但它还是需要一些特殊的逻辑来处理取消订阅以及请求派发。
