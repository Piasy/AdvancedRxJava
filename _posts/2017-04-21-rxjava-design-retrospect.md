---
layout: post
title: RxJava 设计回顾
tags:
    - Operator
---

原文 [RxJava design retrospect](http://akarnokd.blogspot.com/2016/03/rxjava-design-retrospect.html){:target="_blank"}。

## 介绍

RxJava 已经发布三年多了，期间也经历了好几次重大的版本变化。在本文中，我将指出一些我个人认为设计和实现过程中的不足之处。

但不要误会，这并不是说 RxJava 不好，或者我知道怎么做得“更好”。这对所有参与其中的人来说都是一个学习的过程，关键是，我们能否从这些问题中吸取教训，在下一个大版本中做得更好。

## 同步取消订阅

在早些时候，RxJava 仿照了 Rx.NET 的架构，Rx.NET 的两大核心接口是 `IObservable` 和 `IObserver`，它们源自 `IEnumerable` 和 `IEnumerator` 接口。（我自己的 Reactive4Java 库也是这样的设计）

在 `IObservable` 中，有一个 `subscribe()` 方法，它会返回一个 `IDisposable`。返回的 `IDisposable` 对象让我们可以取消整个运行的链条。但这一机制有个关键的问题，下面我用一个最简单的响应式程序演示一下：

~~~ java
interface IDisposable {
    void dispose();
}
 
interface IObserver<T> {
   void onNext(T t);
}
interface IObservable<T> {
    IDisposable subscribe(IObserver<T> observer);
}  
 
 
IObservable<Integer> source = o -> {
   for (int i = 0; i < Integer.MAX_VALUE; i++) {
       o.onNext(i);
   }
 
   return () -> { };
};
 
IDisposable d = o.subscribe(System.out::println);
d.dispose();
~~~

如果我们运行上面的代码，它会向控制台打印数字，尽管我们立即对 subscribe 返回的对象调用了 dispose 方法。问题出在哪儿？

问题在于数据源只有在结束循环之后，才能返回 `IDisposable` 对象，但此时已经没有意义了。整个过程都是同步的，因此也就不可能取消了。

尽管 Rx 能很好地处理异步，但在一个典型的流水线中，很多步骤都是同步的，都会受到同步取消这一要求的影响。由于 Rx.NET 比 RxJava 还要年长 3 年，这么明显的问题怎么可能仍存在于 Rx.NET 中？

上面示例中的代码就是著名的 `range()` 操作符，而如果我们在 C# 中执行类似的代码，就会发现并不会打印数字，或者很快停止。秘密就在于 Rx.NET 的 range 操作符默认是在异步的调度器中执行的，所以循环体会在异步线程中执行，而我们可以立即返回一个可以取消的 `IDisposable`。因此，同步取消的问题就被绕开了，但我不知道 Rx.NET 是有意还是无意为之，天晓得。

如果我们看一下 Rx.NET range 的源码，我们会发现更复杂的东西。它利用了递归调度来向 Observer 传递每一个数据。经过测量，在同一台及其上传递 1M 个数据，Rx.NET 的速度只能达到 1M op/s，而 RxJava 能到 250M op/s。

RxJava 在 range 操作符中并未使用任何调度器，因此同步取消的问题就暴露出来了，我们利用了 `Subscriber` 类来解决这个问题。我们可以检查 subscriber 是否仍希望接收数据。我们可以重写上面的例子，在循环中检查 subscriber 的状态，并按需提前退出：

~~~ java
Observable<Integer> source = Observable.create(s -> {
   for (int i = 0; i < Integer.MAX_VALUE && !s.isUnsubscribed(); i++) {
       o.onNext(i);
   }
});
 
Subscription d = o.subscribe(new Subscriber<Integer>() {
    @Override
    public void onNext(Integer v) {
        System.out.println(v);
        unsubscribe();
    }
 
    // ...
});
~~~

上面的 Subscriber 表现和 `take(1)` 一样，在收到第一个事件之后就取消自己。`unsubscribe()` 方法里面会设置一个 `volatile boolean` 标记，这个标记将在 `isUnsubscribed()` 中返回，因此循环就可以退出了。注意，我们仍无法通过 `subscribe()` 返回的 `Subscription` 函数取消订阅，因为 lambda 表达式中的循环在终止条件满足之前都不会退出。

看起来我们并没有很好地解决最初的问题，对吧？但我们已经可以在循环开始前，或者循环过程中取消订阅了，而第二种情况显然需要在另一个线程中实现：

~~~ java
Subscriber<Integer> s = new Subscriber<Integer>() {
    @Override
    public void onNext(Integer v) {
        System.out.println();
    }
    // ...
}
 
Schedulers.computation().schedule(s::unsubscribe, 1, TimeUnit.SECONDS);
 
source.subscribe(s);
~~~

事实上，我们可以在实际订阅之前设置好取消订阅逻辑，这样即便在最复杂的操作符中，我们也能恰当地把取消订阅的操作传播出去了。

此外，上面的结构中还有一层更深的暗示。在我们创建的 Observable 中，lambda 表达式并没有任何东西，但 `Observable.subscribe()` 调用仍会返回一个实际上和传入的 `Subscriber` 参数一样的 `Subscription`。（技术上来说，这个话题涉及到的内容比较多，可以看看 [Jake Wharton 对这个话题的一个精彩演讲](http://jakewharton.com/presentation/2015-11-05-oredev/)）

更进一步：如果我们需要返回什么东西，那我们就不可能彻底地响应式。返回数据就意味着同步的行为，函数就需要产生结果，即便此时它并不能产生这一结果。这时我们就只能阻塞或者 sleep 到实际产生数据的代码执行完毕了。这一点我在[关于 OSGi Asynchronous Event Streams 的文章中](/AdvancedRxJava/2017/03/25/asynchronous-event-streams-vs-reactive/)有讲述。

## Subscriber 的资源管理

`Subscriber` 类允许我们通过 `Subscription` 对象的形式把资源和它绑定起来。当 subscriber/操作符被取消（或者终止）时，这些资源也就被取消了。

这对操作符的开发者当然方便，但也有其代价：内存分配。

我们通过默认构造函数创建 Subscriber 对象时，其内部的 `SubscriptionList` 也会被创建，无论它是否会被关联资源。在上面的例子中，`range()` 操作符并不需要资源管理，因此 `SubscriptionList` 并没有任何作用。

一方面，很多操作符并不需要资源管理，因此创建这个容器就是一种浪费，而另一方面，也有很多操作符需要资源管理，因此很需要这种便利。

此外，如果我们回忆一下 `Subscriber` 的内容，会发现它还有另一个构造函数，它接收另一个 `Subscriber` 对象，这样我们就可以共享内部的 `SubscriptionList` 对象了。显然，这可以减少一些内存分配，但绝大多数操作符都不能共享内部的 `SubscriptionList` 对象，因为这就会取消订阅下游了（请见 [pitfall #2](http://akarnokd.blogspot.hu/2015/05/pitfalls-of-operator-implementations.html)）。因此 `Subscriber` 的结构，相对于操作符编写者的便利之处来说，在性能的角度来说更是个负担。

你可能会想，给操作符编写者方便的工具有什么不对的？我承认 RxJava 之外的操作符应该尽可能多地带来帮助，但我相信，内部的操作符应该从开始就使用性能更好（即便麻烦一些）的方式实现。

我曾好几次尝试解决这个问题，但鉴于 RxJava 1.x 的架构，我很怀疑这个问题能否被解决。幸运的是，Reactive-Stream 的架构以及 RxJava 2.x，通过把资源管理交给了操作符，解决了这个问题，

## Subscriber 的 request() 函数

如果我们看看 `Subscriber` 的实现，我们会发现一个 `protected` 的 `request()` 函数。这让我们可以很方便地发出请求，并保证如果当前已经通过 `setProducer` 设置了 `Producer`，那就把请求转发给它，如果没有 `Producer`，那就积累请求直到 `Producer` 到来。基本上来说，这就是一个内联的 [producer-arbiter](/AdvancedRxJava/2016/07/02/operator-concurrency-primitives-7/)。

有人可能会觉得这个函数的实现为请求管理带来了很大的开销，但 JMH benchmark 确认了它的影响在 +/- 3% 之内，而误差都可能会导致这种规模的差异。

真正的问题在于它的名字和 `Producer.request` 一样，这就使得我们在继承 `Subscriber` 时无法实现 `Producer` 接口了。

这一问题带来的后果就是，如果主 `Subscriber` 需要做请求管理，就需要一个额外的 `Producer` 对象。

这会导致短序列在订阅时会对 GC 造成较大的影响。另一个影响是这会增加调用栈深度，这可能会阻碍一些 JIT 优化。

由于 `Subscriber.request()` 也是公开 API 的一部分，所以在 1.x 中我们没法把它重命名进而为 `Producer.request()` 腾出空间。

同样，解决方案也在 2.x 中：在 Reactive-Stream 中，`Subscriber` 和 `Subscription` 都是接口，它们可以同时出现，此外，`request()` 函数的便利可以转移到一个方便的 `Subscriber` 实现中（例如 `AsyncSubscriber`），这样就不会影响操作符的内部实现了。（这也意味着我们不鼓励在操作符中利用方便的 `Subscriber`）

## lift

和 backpressure 一起，`Observable.lift()` 被很多人认为是对 RxJava 最好的一个补充。它让我们可以深入订阅的过程中，利用下游的 Subscriber，返回一个新的 Subscriber 给上游，在其中执行操作符的逻辑。

lift 非常流行，现在几乎所有的操作符都使用了它。

不幸的是，有得必有失：内存分配。对绝大多数操作符来说，应用这一操作符会增加 3 个额外的对象分配。为了展示这一点，让我们展开 map 的实现：

~~~ java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    OperatorMap<T, R> op = new OperatorMap<T, R>(func);
    return new Observable<R>(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber<? super R> child) {
            Subscriber<? super T> parent = op.call(child);
            Observable.this.unsafeSubscribe(parent);
        }
    });
}
~~~

我们每次都会创建了一个 Operator 对象，一个 Observable 对象，以及一个 OnSubscribe 对象。

直接使用 map 时这可能不是什么大问题，但想象一下在一个百万次的循环中，每次都有 3 个额外的分配会有怎样的影响？在 flatMap 中如果我们要使用 map 就会出现这样的情况：

~~~ java
Observable.range(1, 1_000_000).flatMap(v -> 
    Observable.just(v).observeOn(Schedulers.computation()).map(v -> v * v))
.subscribe(...);
~~~

lift 操作符实际上就是一个 OnSubscribe 对象，它会捕获上游的 Observable，并对下游的 `Subscriber` 调用 `Operator.call`。显然我们可以直接用 OnSubscribe 实现操作符，并把上游的 Observable 作为参数，这样尽管对象的实际大小并没有多大变化，但内存分配和调用栈都会减少。

当前 lift 的结构还有一个不利的影响：它让操作符融合从困难变成了不可能。因为它是一个异步的类，我们很难获得上游的 Observable 和 Operator；而且即便我们创建了一个命名类，这两个类对象都无法直接获得，要取得它们会带来更多的额外开销。

幸运的是，这里提到的问题我们都可以在不影响公开 API 的前提下解决，只不过需要多写以及多审阅数千行的代码。

不幸的是，我在去年九月实现 RxJava 2.0 预览版的时候并没有考虑过 lift 的开销，因此 2.x 也大量使用了 lift。

不过，在这条道路的尽头还是有光明的：Reactor 2.5 并没有走上 lift 这条路，现在它的开销比 RxJava 小。

## create

最近我开始公开反对 `Observable.create()`，现在我认为我们应该为它改一个更可怕的名字，这样初学者就会避免使用它，进而寻找更合适的工厂方法来创建 Observable 了，以更好地处理 backpressure 和取消订阅。我们可以把它看做一个向听众展示如何进入响应式编程世界的工具，但它确实不应该在演讲中获得这么多关注。

除了这些，create() 的问题还在于它会为每个 Observable 创建两个对象：Observable 对象本身，以及容纳订阅逻辑的 OnSubscribe 对象。

通过 create() 创建 Observable 的做法源自“组合优于继承”的倡导。从普适的设计原则角度来讲，这种做法是没问题的，但我们在 Java 的世界里面需要意识到，组合就意味着内存分配：外部类对象，内部类对象，已经“内部的内部”类对象。

为了避免这些内存分配，解决办法就是让 Observable 不把使用 OnSubscribe 对象作为默认选项（但可以把 create() 保留着，作为一个 lambda 工厂的形式），而且操作符（包括源头中间的操作符）应该直接继承自 Observable。所有的操作符函数都继续保留在 Observable 类中：

~~~ java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return new ObservableMap<T, R>(this, func);
}
~~~

这样，没有了 lift，create 之后，map 只需要每次分配一个 Observable 对象即可。

我相信这些改变并不会影响到公开的 API，因为 Observable 的函数是 static 或者 final 的，而操作符则都是 Observable 的子类。这一改变也有利于操作符融合，因为每个上游都可以直接区分开来了，它们的参数也可以直接暴露出来了。

这里 Reactor 2.5 再次领先了 RxJava，它没有使用 create。它的操作符都是通过继承 [Flux](https://github.com/reactor/reactor-core/blob/master/src/main/java/reactor/core/publisher/Flux.java) 实现的。

## 总结

设计和实现 RxJava 的各个版本一直都是一个学习的过程，而且也会出现一些影响性能和复杂度的意外效果。

你可能会想，为什么这些结构上的麻烦以及内存分配的开销，在当前的状态下运行着？两个原因：云计算和 Android/IoT。在云计算领域，事件都数以十亿计，任何性能的开销都会被急剧放大。你可能无法轻易地计算出上面 flatMap 的例子在笔记本上面的开销，但云计算是按秒，按 GB，GHz 计费的。而对 Android 和 IoT 来说，设备的资源限制以及越来越多的需求要求我们把内存占用、GC 以及电池消耗都考虑进来。
