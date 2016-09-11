---
layout: post
title: Reactive-Streams API（一）：概览
tags:
    - Scheduler
---

原文 [The Reactive-Streams API (part 1)](http://akarnokd.blogspot.com/2015/06/the-reactive-streams-api-part-1.html){:target="_blank"}

## 介绍

在我（_原作者_）个人看来，[Reactive-Streams API](https://github.com/reactive-streams/reactive-streams-jvm/){:target="_blank"} 是目前描述异步、取消、支持 backpressure 数据流时最优雅的 API 了，它受到了 RxJava 很多影响，替换了由 Erik Meijer 提出的 `IObservable`/`IObserver` 设计。

回到 4 月份，我花了一周的时间，尝试把 RxJava 1.x 移植到新的 API 上来。一开始，最大的障碍是新的 `Subscriber` 接口没有资源管理的途径，以及如何把 RxJava 的 `Producer` 和 `Subscription` 合并为新的 `Subscription` 接口。然而，当我开始实现一系列基础支持的类之后，我开始觉得新的 API 简直设计得太巧妙了，而这一设计最重要的收获应该就是 backpressure 的支持一下子变得直截了当了。

带着新的见解，我回过头来看 RxJava 的一些操作符以及 `Producer` 的实现，我突然可以轻易发现一些 bug 以及优化空间了，更不用提后来编写的这一系列讲解 RxJava 工作原理的文章了。

在这个小的系列中，我将介绍新的 reactive-streams API，以及它与 RxJava 中一些基础结构的关系。

## 包的组织，以及命名上的困惑

熟悉了 RxJava 之后，面对 reactive-streams（RS）的第一头拦路虎应该就是同样的概念被取了不同的名字。

和 `rx.Observable` 对应的是 `org.reactivestreams.Publisher`，而且它是一个接口。由于 Java 不支持而且很可能也不会支持扩展方法，所以用一个 `Publisher` 接口返回数据源，对于在它上面应用一系列操作并没有什么帮助。幸运的是，通过继承新的接口，目前 RxJava 已有的实现都可以保留。

用于取消的 `rx.Subscription` 和 backpressure 控制的 `rx.Producer` 现在被合并为了 `org.reactivestreams.Subscription` 接口。而且取消的接口也从 `unsubscribe()` 变成了 `cancel()`，而且也没有（也几乎不需要）检查是否已经取消的 `isUnsubscribed()` 接口了。由于 RS 没有规定任何资源管理的方式，我们需要把 `cancel()` 方法包装到一个资源管理的对象中，使得我们可以在容器中使用它。由于这一命名的困惑，RxJava 2.0 很可能会把资源管理相关的类改成 C# 风格的 `Disposable` 接口。

RS 中没有一个和 `rx.Observer` 对应的存在，但 `org.reactivestreams.Subscriber` 和 `rx.Subscriber` 比较类似。新的 Subscriber 也不能直接被取消，它将接受一个控制对象，用来执行取消操作，或者请求更多操作。此外，它也不再保留 `onStart()` 和 `setProducer()` 这两个方法，这两个方法在 RxJava 中常常导致大家不知道该怎么启动一个 `Subscriber`，它只会有一个接收 `Subscription` 的 `onSubscribe()` 函数。前面的方式会调用 `rx.Subscriber.request(n)` 函数，而后面的方式则需要保存 `Subscription` 对象，并调用 `org.reactivestreams.Subscription.request(n)`。

还有人对终止信号的函数名称存在一些争议，RS 中它叫做 `onComplete()`，而不是 RxJava 以及 Rx.NET 中的 `onCompleted()`。希望 IDE 自动补全的功能可以减少我们在这一变化上的问题。

最后，RS 定义了一个名为 `Processor` 的接口，它继承了 `org.reactivestreams.Publisher` 和 `org.reactivestreams.Subscriber`，它和 RxJava 中的 `Subject` 一样。由于 `Processor` 是一个接口，所以对 `rx.Subject` 唯一的更改就是实现这个接口。

## Java 9+ Flow

RxJava 和 RS 之间还是有些不一样的，但你可能听说了 Doug Lea 希望把 reactive-streams 的思想引入 Java 9，利用一个叫做 `java.util.concurrent.Flow` 的容器类。名字和 RS 一样，但包名不一样。我对此一直持怀疑态度：

+ 这个“通用的平台”并没有那么通用：只有 Java 9+ 的用户才能使用。RS 对 Java 6 兼容以及安卓平台友好，是一种更好的选择。绝大多数项目都引入了很多第三方库，不差多 2 个（RS + RxJava 2.0）。
+ Java 9 不会在内部使用这套 API。目前没有看到任何基于 Observer 模式的 NIO，网络，文件 I/O，以及 UI 事件系统的改进计划。更何况，如果没有流畅的 API 支持，绝大部分用户依然需要额外的依赖才能使用。通过对老版本的 JDK 发布一个包含 Flow 的更新，可以缓解这一情况。但是，没有方法扩展的话，用户仍然不得不依赖额外的库，以及实例包装。
+ 我并不确定响应式编程已经确定了它的最终形态：例如，Applied Duality 的 `AsyncObservable` 是用来捕获老的 `onXXX` 事件的，将来有一天可能会被合入 RS 2.0（尽管我不认为有这样的需求）。把依赖改成 RS 2.0 和 RxJava 3.0 也许比改变 JDK 的 Flow API 更加简单。这里有一个历史教训，Rx.NET 把 `IObservable`/`IObserver` 加入到了 BCL 中，我认为这是给自己挖了个坑，如果想要加入 RS 阵营，他们就需要想出怎么让新老接口能一起保留，就像一个集合需要考虑前置泛型和后置泛型一样（pre-generics and post-generics collections）。注意这里我假设他们接受 RS 作为 Rx.NET 的基础，而不是再发明一套新的东西。

## 对 RS 的一些评价

我对 RS 新的 4 个接口非常满意，但其文字规范有点差强人意。我认为它太过严格，而且会妨碍一些很明智的实现方案。

### 只允许 NullPointerException

尽管函数允许在传入参数是 null 时抛出 NPE，但是各个模块可能会处于非法的状态，或者 `Subscription.request()` 会被传入非法的负参数。目前 RS 1.0 的规范不允许按照大多数 Java API 的方式处理这些情况，即抛出 `IllegalArgumentException` 和 `IllegalStateException`。

有人可能会说可以利用 `onError()` 来发出这些异常，但我会展示一些特殊情况，如果我们严格遵循 RS 规范，我们就无法处理。

假设我们有一个 `Subscriber`，而它错误地订阅了两个不同的 `Publisher`。规范中定义，这种情况是不允许的，应该在 `onSubscribe()` 中拒绝第二个 `Subscription`：

~~~ java
// ...
Subscription subscription;
@Override
public void onSubscribe(Subscription s) {
    if (subscription != null) {
        s.cancel();
        onError(new IllegalStateException("§x.y: ..."));
        return;
    }
    this.subscription = s;
    s.request(Long.MAX_VALUE);
}
~~~

看起来没什么问题，但是第一个 `Publisher` 是异步的，这就会导致两个问题：

+ 第二次 `onSubscribe()` 调用立即执行了 `onError()`，这就可能会和第一个 `Publisher` 的 `onNext()` 同时发生。而这是规范禁止的。
+ 不能直接保存 `Subscription` 的引用，需要使用 `AtomicReference` 以避免竞争，二者又增加了 `Subscriber` 的实现开销。

所以唯一可能的 `onSubscribe()` 安全实现方式只能像下面这样：

~~~ java
// ...
final AtomicReference<subscription> subscription = ...;
@Override
public void onSubscribe(Subscription s) {
    if (!subscription.compareAndSet(null, s)) {
        s.cancel();
        new IllegalStateException("§x.y: ...").printStackTrace();
        return;
    }
    s.request(Long.MAX_VALUE);
}
~~~

并且依赖打印的日志被监控，然后串行化调用 Subscriber 的 `onXXX()` 方法。

但我觉得直接抛出 `IllegalArgumentException`，让 `subscribe()` 调用失败是最简单的办法。

### 非法的请求数量

`Subscription.request()` 应该传入一个正数，否则就应该通过 `onError()` 通知 Subscriber：

~~~ java
// ...
@Override
public void request(long n) {
    if (n <= 0) {
        subscriber.onError(new IllegalArgumentException("§x.y: ..."));
        return;
    }
    // ...
}
~~~

同样，如果我们异步调用的 `request()`，例如 `observeOn()` 希望在通过 `onNext()` 发出数据的同时，补充输入队列。这时 `onError()` 和 `onNext()` 也可能并行发生，这也是禁止的。

我的建议和处理 `IllegalStateException` 类似：我们只能打印日志，不能直接抛出异常。

### `request()` 不能抛出 NPE

即便 `onXXX()` 可以抛出 `NullPointerException`，但 `Subscription.request()` 不能。我们只能把异常“反弹”（_没懂啥意思_）给 `Subscriber`，而如果这也失败了，那就只能打印日志了。

### `Processor` 必须要一个 `Subscription`

RS 规范严重倾向于 cold observable，而 cold observable 必须有 `Subscription`。然而像 RxJava 中 `Subject` 这样的 hot observable，在 `onNext()` 被调用之前是有可能没有 `Subscription` 的。

例如，如果你调用 `Observable.subscribe(PublishSubject)`，用 `PublishSubject` 来多播（multicast）一个 `Observable`，`PublishSubject` 会得到一个 `Subscription`。但如果用 `PublishSubject` 来多播鼠标移动的事件，这时是没法取消（以及支持 backpressure）的，因此 `Subscription` 是不必要的。

这时我们就不得不用一个假的 `Subscription`，或者让 `onSubscribe()` 的调用成为 `Processor` 的一个可选项。

### `Processor` 必须支持 backpressure

backpressure 的支持对 `Processor` 来说有时是不可能的：例如鼠标移动事件，是无法支持 backpressure 的，因此 `Subject` 可能需要缓冲或者丢掉过多的数据。把它实现为 `AsyncSubject` 是很琐碎的，无需考虑，但是如果使用其他的类型，就不得不转化为某种 `ReplaySubject`（很自然地，`PublishSubject` 会在没有收到 subscriber 时丢掉未请求的数据，但是持续传递数据的保证在我看来是非常有价值的，无法被推翻）。

如果要遵守这一规范，将给所有的 subscriber 带来很大的开销，数据必须经过队列，而且可能需要使用锁，使得处理受限于最慢的请求者。

如果是我，我会放宽规范，让 backpressure  的支持成为 `Processor` 的一个可选项。

### `Subscription` 将会变得更保守

规范中建议，`Subscription` 在实现时可以假定并发只是少数情况。如果你还记得 RxJava 中 `Producer` 的要求，我提到的是线程安全和可重入安全。

在 `Subscription.request()` 的实现中，可重入安全依然是非常关键的。它能避免不必要的递归，而且也能处理 `onNext()` 中在处理当前数据之前调用 `request()` 的情况，这种情况下又可能会触发 `onNext()`，然后就陷入了无限递归。

线程安全的需求可以被以下链条的行为证明。假设我们有一个链条在主线程订阅，不使用 RxJava 的 scheduler，通过 `observeOn()` 改变接收数据的线程。数据的消费者持续调用 `request()`，而这些调用都会转发到生产数据的数据源那里（主线程）。这时，就会有两个线程处于或者进入 `request()` 函数（_how?_），而且没有相应的原子性、同步保证，则重入检查以及请求处理将会出问题。有人可能会认为 `observeOn()` 已经把对 `Subscription` 的访问串行化了，但由于数据源并没有在 `Scheduler` 中执行，这些串行化在下游看来是没有作用的，而从上游来看则是低效的。

由于 Subscriber 不知道 `request()` 的调用特性，它就不得不变得很保守，利用我在前文中介绍 `Producer` 时提到的串行访问方式。很自然地，任何对 `cancel()` 的实现，都不得不利用一个 `volatile` 变量了。

## 总结

上面的这些考虑并不是在写这篇文章的时候产生的，我早就尝试告知 RS 的设计者们了。不幸的是，我的考虑并没有被他们接受，而我的代码也没能赢得辩论。我们可以无休止地讨论我提出的问题，但我更倾向于在这里用代码来证明我的观点。

不管怎样，如果 RS 能进行以下几点修正，它将会是一个非常好的设计规范：

+ 错误处理：引用一下最新的 Godzilla 电影，“Let them throw！（抛出去）”
+ `Processor` 可以选择在 `Subscriber` 端实现 backpressure 和取消。
+ 绝大多数 `Subscription` 都应该更保守一点，并且实现线程安全。

在接下来的文章中，我将从 RxJava 中的两个简单 `Producer` 入手，`SingleProducer` 和 `SingleDelayedProducer`，展示如何把它们转换为 `SingleSubscription` 和 `SingleDelayedSubscription`。
