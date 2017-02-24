---
layout: post
title: 深入理解 Operator：AutoConnect
tags:
    - Operator
---

原文 [Operator internals: AutoConnect](http://akarnokd.blogspot.com/2015/10/operator-internals-autoconnect.html){:target="_blank"}。

## 介绍

`autoConnect` 操作符是 `ConnectableObservable` 类的一部分，它允许我们在一定数量的 Subscriber 订阅（返回的 Observable）之后，自动连接原来的 ConnectableObservable。这个操作符返回的就是一个普通的 Observable，因此我们可以更容易地和其他操作符串联起来。

设计这一操作符有两个初衷。首先，我们有时候希望只有当一定数量的 Subscriber 订阅之后才连接 ConnectableObservable，没有 autoConnect 我们很难实现这一功能；其次，另一个 cache 操作符不支持高级的留存策略控制，例如按时/按量控制缓存的数据量，而实现像 cache 这样首次订阅触发连接的逻辑又比较繁琐（_所以干脆实现一个新的操作符，同时具备留存控制和自动触发功能_）。

总的实现思路很简单，我们用一个 AtomicInteger 在 OnSubscribe 类中记录抵达的 Subscriber 数量，一旦到达目标数量，我们就可以调用 connect 了。

但这里有一个细节比较复杂：ConnectableObservable 同步取消订阅的支持。如果使用上面实现的 autoConnect，我们就无法取消订阅上游数据流了，这和 cache 的行为很相似。解决办法就是准备一个回调，把它传给 connect 函数。

## 实现细节

由于这个操作符不需要请求管理，因此 1.x 和 2.x 的实现基本一样：

~~~ java
public Observable<T> autoConnect(int numConnections,
        Action1<Subscription> connection) {               // (1)
    if (numConnection == 0) {
        connect(connection);                              // (2)
        return this;
    }
    AtomicInteger count = new AtomicInteger();
    return create(s -> {
        unsafeSubscribe(s);                               // (3)
        if (count.incrementAndGet() == numConnections) {  // (4)
            connect(connection);
        }
    });
}
~~~

有几点还是值得一提的：

1. RxJava 还有两个重载版本，一个不需要 `numConnections` 参数，默认为 1，另一个不需要 `connection` 参数，默认什么也不做；
2. 如果 numConnections 为 0，我们直接连接 ConnectableObservable，这种情况下我们不需要做任何包装，可以直接返回 this；
3. 如果 numConnections 不为 0，那我们就记录接下来到达的 Subscriber，并在计数到达预定数量之和，连接 ConnectableObservable；但在检查计数之前，我们先把 Subscriber 订阅到 ConnectableObservable 上；先订阅这一点很重要，因为如果 numConnections 为 1，那连接操作可能会导致数据的发射，而如果后订阅，那实际的 Subscriber 就收不到数据了；
4. 一旦 Subscriber 计数到达预定数量，我们就连接 ConnectableObservable，这会导致触发最初提供的回调；

此时你可能会想，如果 numConnections 为 2，第一个 Subscriber 订阅之后立即取消订阅，那第二个 Subscriber 是否应该触发连接？这取决于我们的需求。autoConnect 显然没有处理这一问题（_也就是说会触发连接_），在取消订阅时调用 decrementAndGet() 就可以处理这一问题。

这么处理的原因是 autoConnect 最初是用来在特定情况下替代 refCount() 和 share() 的。

## 总结

autoConnect 可以说是前 10% 简单的操作符了，所以它的特性和实现都很简洁明了。
