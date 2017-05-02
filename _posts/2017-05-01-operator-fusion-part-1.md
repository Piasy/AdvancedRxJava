---
layout: post
title: 操作符熔合（一）
tags:
    - Operator
---

原文 [Operator-fusion (Part 1)](http://akarnokd.blogspot.com/2016/03/operator-fusion-part-1.html){:target="_blank"}。

## 介绍

操作符熔合是响应式编程领域最尖端的研究话题之一，它的目的是把数据流中使用的多个操作符以某种方式结合起来，进而降低开销（时间，内存）。

（_其他尖端的话题包括：1. 响应式 IO；2. 更多原生的异步并行序列；3. 透明远程查询（transparent remote reactive queries）。_）

操作符熔合最关键的出发点有三个：

1. 很多序列的数据都是从不变的（或者类似不变的）数据源发出的，例如 `just()`，`from(T[])`，`from(Iterable)`，`fromCallable()`，它们在应用一系列操作符时并不需要考虑线程安全性（_因为是单线程_）；
2. 有些操作符可以共享一些内部组件，例如内部队列；
3. 有些操作符可以表明是消费了数据还是丢弃了数据，因此可以省去 `request(1)` 调用的开销；

在这个短系列中，我将基于我们的理解，讲讲怎样实现操作符熔合，以及为什么要实现操作符熔合。这里的“我们”，是指在 RxJava 2.x 以及 Project Reactor 早些版本中对 Reactive-Streams 操作符的共同优化研究。

实验代码在 [Reactive-Streamss-commons](https://github.com/reactor/Reactive-Streamss-commons) 仓库中，简称 **Rsc**。Rsc 的结果驱动着 Project Reactor 2.5（milestone 2），并且经大量用户验证。希望 RxJava 也可以从中获益（但可能在 3.x 之前都无法实现）。

如果你一直关注着 Akka-Streams，那可能也看到或者听到了它也有操作符熔合。不过[据我对它的原理的理解](http://doc.akka.io/docs/akka/snapshot/java/stream/stream-flows-and-basics.html#Operator_Fusion)，它的主要目标是确保流水线中尽可能多的环节发生在同一个 Actor 中，避免此前很可能存在的线程跳跃（thread-hopping）问题。此外，现在他们还有一种模式，允许开发者自定义流水线中异步的边界（_允许定义哪些代码异步执行，哪些代码同步执行_）。是不是听起来很耳熟？基于 Rx 的库从最开始就支持这个特性（_`subscribeOn` 和 `observeOn`_）。

## 分代

响应式编程的库及其相关概念一直在进化。7 年前我们在 Rx.NET 中的要求以及实现方案，和我们将来像 Project Reactor 这样的库相比，肯定迥然不同。

根据我对“现代”响应式编程历史的经验，我把各个库分为不同的代。

### 第 0 代

最初的响应式编程工具主要由 `java.util.Observable` API 以及它在其他语言中的表亲构成，甚至包括所有基于回调的 API，例如 Swing/AWT/Android 中的 `addXXXListener`。

Observable API 最可能源自四人帮的设计模式书（或者其他类似的来源，谁知道呢），它的不足之处在于使用不便，且无法组合。从当今的角度来看，它就是一个受限的 `PublishSubject`，我们只有一个阶段：publisher-subscriber。

尽管 `addXXXListener` 风格的 API 便利了基于 push 的事件处理，但它却难以组合使用。缺乏基本可组合概念，使得我们只能逐个实现可组合的库；或者我们设计一套类似于 RxJava 的通用抽象层，然后为每个 `addXXXListener`/`removeXXXListener` 进行适配。

### 第 1 代

一旦难以组合的问题被 Erik Meijer 和他在 Microsoft 的团队发现并解决之后，第一代响应式编程的库诞生了：2010 年左右的 Rx.NET，2011 年的 Reactive4Java，以及 2013 年早期的 RxJava。

其他的库紧密追随了 Rx.NET 的架构，但很快发现了这种架构的问题。当最初的 `IObservable`/`IObserver` 在纯单线程中实现后，应用了诸如 `take()` 操作符之后的序列，根本无法取消。Rx.NET 通过在诸如 `range()` 的数据源强制异步，绕开了这个问题。

第二个问题发生在 Producer 和 Consumer 之间存在一个异步边界（_两者不在同一线程_），而 Consumer 速度不够快的情况下。这种情况下我们的 Consumer 代码也非常繁琐，因为库的基础设代码在跨异步边界时的开销。这就是我们所说的 backpressure 问题。

### 第 2 代

新架构中的同步模式取消问题，以及缺乏 backpressure 支持的问题被 RxJava 团队发现了（但我实际没有参与多少），于是他们设计了一套新的架构。

我们引入了 `Subscriber` 类，它通过 `isUnsubscribed()` 函数表明自己是否仍对新数据感兴趣，数据源或者操作符发数据之前都需要调用该函数检查。

backpressure 的问题则通过双方协调的方式解决，利用 `Producer` 接口，Subscriber 告知上游自己能处理数据的量（_`request()` 函数_）。

第三个新增的部分就是 `lift()` 函数，它允许我们直接在 Subscriber 之间进行函数式的变换。几乎所有的实例操作符都被重写，改成了利用新的 `Operator` 接口和 `lift()` 函数。

### 第 3 代

除了[稍显笨拙以及限制了一些优化方案](/AdvancedRxJava/2017/04/21/rxjava-design-retrospect/)之外，RxJava 的另一个问题是和其他（将要发布的）响应式变成库无法兼容。意识到响应式编程（支持 backpressure）的兴起之后，来自各个公司的工程师们聚在了一起，设计了一套 Reactive-Streams 规范。它的主要成果是 4 个接口，30 条关于这几个接口的规则，以及这几个接口里的 7 个方法。

Reactive-Streams 规范使得响应式编程实现库之间可以相互兼容，可以跨越各个库的边界，组合序列、取消序列以及实现 backpressure，并且还能随意切换具体的实现库。

因此 Reactive-Streams 属第 3 代，它的实现包括 RxJava 2.x，Project Reactor 和 Akka-Streams。

### 第 4 代

在 Reactive-Streams 之上实现一套可组合的库需要完全不同的内部架构，因此 RxJava 2.x 不得不完全重写。我在重写的过程中，发现有些操作符可以通过某种方式进行合并（无论是在 RxJava 内部还是外部），以节省各种开销，例如队列，并发原子操作，以及数据请求。

由于有些团体对这个话题不太感兴趣，RxJava 2.x 的开发不得不收尾，所以直到 Stephane Maldini（Reactive-Streams 的贡献者，以及 Project Reactor 的主要贡献者）和我开始讨论一些 RxJava 2.x 和 Project Reactor 2.5+（以及 Akka-Streams）可以使用并收录的基础操作符，我一直没有在 RxJava 2.x 中实施这些想法。

经过积极的交流之后，我们创建了一个 reactive-streams-commons 库，实现了这些基础的操作符，并设计了一套实现上述优化的组件，现在我们称之为**操作符熔合**。

因此，第 4 代的响应式编程库和第 2 代从外部看起来可能很像，但其实内部操作符的实现发生了很大的变化，以支持节省开销，以及更多的可能。

### 5 代之后

我认为现在（_16 年 3 月_）我们已经到达了操作符熔合目标的中点，但我们已经发现了需要扩展 Reactive-Streams 的信号，以支持双向序列（或者管道）形式的响应式 IO。此外，透明远程查询也可能需要 Reactive-Streams 做些变化（可以参见 Rx.NET 中的 QBservable）。我现在还没有看到全部的可能性以及要求，一切都还处于开放讨论中。

## Rx 的生命周期

在开始操作符熔合之前，我想先定义 Rx 序列生命周期的几个主要节点（以及我们将要使用的术语）。它们适用于任何 RxJava 的版本，也适用于其他任何基于 Reactive-Streams 的库。

生命周期可以被分为三块：

1. **装配时（Assembly-time）**。这是我们编写了诸如 `just().subscribeOn().map()` 的代码，并把结果赋值给 `Observable/Publisher` 类型的成员或者局部变量的时期。这是和基于 Future API 的库（`Promise`，`CompletableFuture` 等）的主要区别，如果它们要支持可组合的 API，它们不会有明确的装配时，而是某种横跨三个时期的形式。
2. **订阅时（Subscription-time）**。这是我们准备了一个 Subscriber 并订阅到最终序列上时，它会触发一系列操作符内的“订阅风暴”。一方面是向上游调用 `subscriber()`，另一方面是对下游调用 `setProducer`/`onSubscribe`。这是订阅副作用触发的时机，此时流水线（pipeline）中尚没有任何数据流动。
3. **运行时（Runtime）**。这是数据被产生并发往下游，且以零个或者一个终止事件（onError 或者 onComplete）结尾的时期。

每个不同的时期都有不同的优化可能。

## 操作符熔合

我承认，我从 Intel CPU 文档中借鉴了一些名词，它听起来很酷，而其背后的概念可以在语言层面展开，并实现到响应式数据流的操作符中。

### 宏熔合（Macro-fusion）

宏熔合主要发生在装配时，它主要是把连续的多个操作符合并为单个操作符，进而优化序列订阅时的开销（有时由于 JIT 过劳，因此也能优化运行时开销）。有以下几种方式实现宏熔合。

#### 1) 把操作符替换为另一个操作符

在这种形式的熔合中，被应用的操作符查看上游数据源，按需调用/初始化另一个操作符，而不是实例化自己的实现（这也是我[之前提到 `lift()` 会导致问题](/AdvancedRxJava/2017/04/21/rxjava-design-retrospect/)的原因）。

例如我们把 `amb()`/`concat()`/`merge()` 应用到一组数据源，但它们只会发出一个数据时，我们就没必要初始化这些操作符，直接返回这个数据即可。这类优化在 RxJava 1.x 中就已经实现了。

第二个例子是当我们使用固定不变的数据源（例如 `range()`）并应用 `subscribeOn()` 时。在这种情况下应用 `observeOn()` 和 `subscribeOn()` 的表现微乎其微，所以如果 `subscribeOn()` 检测到上游是 `range()`，就会换成应用 `observeOn()`，寄希望于 `observeOn()` 能带来其他优化。

#### 2) 把操作符替换为自定义操作符

有些操作符常常成对出现，而把它们合并为单个操作符可能会带来性能提升。最常见的操作符对就是用来开启异步执行的 `just().subscribeOn()` 或者 `just().observeOn()`（_这俩效果一样_）。

这种序列的开销相较于它们发出的单个数据，就非常明显了：内部队列的创建，调度器 worker 的创建和销毁，多个原子变量的修改。

因此把这种操作符对合并为单个操作符，把调度和发出这个唯一的数据这两件事结合起来，就能带来显著的优化了。

上面这种方式可以被扩展到其他的操作符中，尤其是涉及到 `just()` 时，例如 `flatMap()`，所有[内部的复杂度](/AdvancedRxJava/2017/04/10/flatmap-part-1/)都可以跳过，我们直接用这个数据调用 mapper，并使用这个单一的 `Observable`/`Publisher` 即可，无需额外的缓冲和同步。

同样的，RxJava 1.x 也已经应用了这些优化。

#### 3) 在订阅时做替换

有时候前面提到的两种情况会发生在订阅时而不是装配时。

我觉得把优化移到订阅时有两个原因：1) 防止 fluent API 被 bypass 的安全网（_实在不懂是什么意思_）；2) 如果熔合前后版本的差别不够创建单独的操作符实现类，那在订阅时做优化会更方便。

#### 4) 替换为同一个操作符，但修改参数

开发者通常会多次应用同一个操作符，例如 `map()` 和 `filter()`：

~~~ java
Observable.range(1, 10)
   .filter(v -> v % 3 == 0)
   .filter(v -> v % 2 == 0)
   .map(v -> v + 1)
   .map(v -> v * v)
   .subscribe(System.out::println);
~~~

这种写法看起来很清晰，但如果我们的 range 长度是 1M，或者反复订阅这个序列百万次，这种写法就会带来很明显的性能开销了。

这类宏熔合的思路是检查是否同类操作符此前已经应用过了，如果是，那就把应用前的数据源取出来，把同类操作符的参数组合起来，一次应用。在上面的例子中，`range()` 后面只会应用一个 `filter()`，它会把两个 lambda 表达式合并起来：

~~~ java
Predicate<Integer> p1 = v -> v % 3 == 0;
Predicate<Integer> p2 = v -> v % 2 == 0;
 
Predicate<Integer> p3 = v -> p1.test(v) && p2.test(v);
~~~

`map()` 也可以进行类似的熔合：

~~~ java
Function<Integer, Integer> f1 = v -> v + 1;
Function<Integer, Integer> f2 = v -> v * v;
 
Function<Integer, Integer> f3 = v -> f2.apply(f1.apply(v));
~~~

### 微熔合（Micro-fusion）

微熔合通过多个操作符共用内部资源和数据结构以减少开销，微融合大多发生在订阅时。

微熔合最初的想法，来自于我们发现，有些拥有输出队列的操作符，和那些需要输入队列的操作符可以共用同一个队列实例，这样可以节省内存分配，以及减少漏循环中的 WIP 计数器原子操作。后来这一想法被扩展到可以暴露出队列的操作符中，这样就能更普遍的省去 `SpscArrayQueue` 的创建。

微熔合有下面几种形式。

#### 1) 条件性 Subscriber（Conditional Subscriber）

当我们使用过滤操作符（`filter()` 或者 `distinct()`）时，如果上游使用了队列漏的请求处理，那就很有可能 `filter()` 会在丢掉最后一个数据之后调用 `request(1)`。`request(1)` 会触发原子递增操作，或者是 CAS 循环，大量这样的操作很快就能积累出性能下降。

条件性 Subscriber 的思路是为 Subscriber 增加一个 `boolean onNextIf(T v)` 函数，它可以告知上游自己是否会真的消费这个数据。这种情况下，在漏循环中我们就可以在数据被丢弃时，跳过原子递增，并继续发射数据，直到实际发出的数据量达到了请求数。

这一优化减少了大量请求管理的开销，RxJava 2.x 中已经有一些操作符实现了这一优化，但它也存在不足之处，受影响的大多都是库编写者：

a) 上游和 filter 之间可能被其他操作符隔开，那这些操作符也就需要提供一个 conditional Subscriber 版本，以转发 `onNextIf()` 调用。

b) 由于有了返回值，`onNextIf()` 的实现就必须是同步的。不过由于只是返回一个 `boolean`，它仍可以像常规的 `onNext()` 那样，声称自己消费了数据，但实际上丢掉了这个数据；当然，这是它也必须调用 `request(1)` 了。

由于这只是一个内部设计，支持 conditional Subscriber 的操作符也必须实现常规的 `onNext()` 函数，以防上游不支持条件性发射，或者上游是其他具有不同内部设计的响应式编程库。

#### 2) 同步熔合

同步熔合是指那些操作符的上游必然是同步的情形，它们可以假装自己是一个队列。

通常这样的操作符有 `range()`，`fromIterable`，`fromArray`，`fromStream` 和 `fromCallable`。当然 `just()` 也算，不过它在宏熔合中涉及得更多。

内部使用队列的操作符包括 `observeOn()`，[`flatMap()` 的内部数据源](/AdvancedRxJava/2017/04/15/flatmap-part-2/)，`publish()`，`zip()` 等。

同步熔合的思路就是，上游的 `Subscription` 实现 `Queue` 接口，在订阅时的 `onSubscribe()` 函数中，如果我们发现 Subscription 实现了 Queue 接口，那就无需创建自己的队列。

这需要上游和操作符都实现一种新的操作模式，在这种模式下，不允许调用 `request()`，而且必须在成员变量中记住当前的模式。此外，如果 `Queue.poll()` 返回 null，这意味着以后都不会再有数据了，这和常规操作符的 `poll()` 操作不同，常规操作符中 pull 返回 null 说明当前已经没有数据，以后肯能会有新的数据。

不幸的是，这种熔合只有在 Reactive-Streams 架构下才能更好地发挥作用，在 RxJava 1.x 中不行，因为：a) 设置 `Producer` 是可选的；b) 生命周期相关的行为不可依赖；c) 难以发现这种情况，以及存在太多的间接关系。

在 Rsc 的基准测试中，这一熔合方案让 `range().observeOn()` 的吞吐量从 55M Op/s 提升到 200M Op/s，在这个简单的数据流中带来了近 4 倍的提升。

同样的，这种 API 级别的 hacking 也存在其不足之处：

a) 在短序列中，操作符的模式切换可能并不划算。

b) 这一优化是库内局部的，除非有 Reactive-Streams 这样的标准 API，A 库中的微熔合很可能无法和 B 库熔合。

c) 在有些情况下这种队列熔合的优化是无效的，主要是由于线程边界的违背（或者其他我们尚未发现的会创建无效熔合序列的情况）。

d) 这一优化会影响到库的其他部分，因为中间操作符需要支持模式转换，或者至少不影响模式切换的设置。

e) 在 Reactive-Streams 架构下，实现了这一优化的操作符将无法直接把上游的 Subscription 传给下游，因为如果下游实现了这一熔合，那中间操作符就被短路了。

#### 3) 异步熔合

在一些情况下，数据源也有自己的队列，会在下游发出请求时漏出，但时间和数量都无法提前得知。

在这种情况下，数据源也可以实现 Queue 接口，操作符就可以直接使用它了，而不用创建自己的队列，但它们之间的协议必须改变，尤其是这个操作符也支持同步熔合时。

因此，在 Rsc 中我们并不是在 onSubscribe 中检查 Subscription 是否实现了 Queue 接口，而是创建了一个新的接口 `QueueSubscription`，它实现了 Subscription 和 Queue 接口，并加上了一个 `requestFusion()` 函数。

`requestFusion()` 接收一个 int 标志，用于告知上游，操作符需要或者支持哪种熔合模式，而上游则返回自己启用了哪种熔合模式。

例如 `flatMap()` 向它的内部上游请求同步熔合，而内部上游则可能返回不支持、支持同步熔合、支持异步熔合，并根据自己返回的模式运行。通常，同步熔合都可以“降级”为异步熔合或者不熔合，但无法从异步熔合“升级”为同步熔合。

在异步熔合模式下，下游仍需要调用 `request()`，但产生的数据不会被两次放进队列，而是被放进共享队列中，同时上游调用 `onNext()` 来通知下游。onNext 的值不重要，我们用 null 来作为类型无关的数据，用来直接出发 `drain()` 函数。

由于熔合发生在订阅时，所以我们已经来不及替换 `Subscriber` 对象了，因此操作符中需要保存一个模式标志，并在后续检查是否处于异步熔合模式。这样操作符就能同时支持普通上游以及可熔合的上游了。

这里的复杂度至少是普通 backpressure 操作符的 1.5 倍，需要大量的操作符及其在各种情况下行为细节的知识。

### 无效熔合（Invalid fusions）

我们无法熔合每个操作符的队列，因为存在无效熔合的情况。

操作符都有自己的某些边界，这些边界和内存边界很像，也有类似的效果：1) 不允许有些重排序；2) 不允许某些优化的组合。

例如，把 String 映射到 Integer，再映射到 Double，就不允许重排序，因为类型不匹配。重排序 `filter()` 和 `map()` 操作也可能是无效的，因为 map 可能会改变类型，而且可能会导致副作用，因为 filter 可能会丢弃掉有问题的数据。

一方面，这些功能性边界大多会影响到宏熔合，而且也容易发现和理解。

另一方面，一旦涉及到异步，例如通过 `observeOn()` 切换线程，那微熔合也可能变得无效。

例如我们有这样一个序列：

~~~ java
source.flatMap(u -> range(1, 2).map(v -> heavyComputation(v))
    .observeOn(AndroidSchedulers.mainThread()))
.subscribe(...)
~~~

内部的 `range-map-observeOn-flatMap` 序列将会使用同一个共享的队列，那 map 就会被重排序到队列的输出端，那就把耗时计算操作移到主线程上了。

_注：通常 `observeOn` 都能把数据的发送“拖到”自己的线程，因为 backpressure 就是这样触发数据发送的，在上面的例子中，如果我们的 range 范围更大，range 操作以及 map 操作最终还是会在主线程执行。因此我们需要在 map 之前使用 `subscribeOn()`/`observeOn()` 以确保 map 发生在我们希望的线程中。_

这要求我们稍微改变一下 `requestFusion()` 的协议，我们需要一个标记位，来表明调用链是否作为异步边界，也就是说，熔合队列的输出端是否会运行在另一个线程。诸如 map() 这样的中间操作符将会拦截这个方法，并返回不支持熔合。

最后，也可能存在订阅时的边界同样会因为订阅时的副作用而组织重排序和优化操作。这一点我们尚未明确，不过有几个例子可以继续研究一下：

1) 把 `range().subscribeOn(s1).observeOn(s2)` 熔合为 `range().observeOn(s2)` 是否有效？我称前者为强流水线序列，因为它存在强制的线程边界切换。“尾发射”模式保留了下来，我们也在 Scheduler s2 上收到事件，但我们丢掉了强流水线效果。

2) 当有大量 Subscriber 时，订阅到 Subject 上可能会带来一些开销，因此使用 `subscribeOn()` 应该可以把开销转移到指定的线程，但通常来说订阅到 PublishSubject 并不会有这样的开销，那我们这时能否去掉 `subscribeOn()` 操作？

## 总结

操作符熔合是降低响应式数据流开销的一个大好机会，我们也有责任这样做，我们可以把开销降低到接近于常规的 Java Stream 序列（Project Reactor 2.5 M1 降低了 50%+，RxJava 2.x 则降低了 200%+），同时还能支持（一部分）异步序列，也能保持 API 不变（内部实现也比较类似）。

然而热情地为每个操作符加上熔合机制也许并不划算，我们应该关注于那些为用户做事最多（_也就是对用户而言最重要的，doing the heavy lifting in user's code most of the time，这句话我特地向原作者请教了，没见过 heavy lifting 这个短语被贻笑大方_）的操作符：`flatMap()`，`observeOn()`，`zip()`，`just()`，`from()` 等等。此外，我们也可以说任何两个操作符都是可以宏熔合的，因为我们可以为任意两个联用的操作符编写一个自定义操作符，但这样就会发生指数爆炸了。

当然，也存在一些操作符看起来不能被（微）熔合，但最后却发现可以熔合。但我们不必为所有的操作符构建一个巨大的交叉熔合矩阵，兴许我们可以为操作符以及数据序列以某种方式进行建模，在模型上通过图算法自动发现那些可以被熔合的操作符，而这，又将是另一个可以研究的话题了。

在下篇中，我将深入展示在 Rsc 中我们是如何实现熔合的，但在此之前，我希望可以先深入讲解一下 `subscribeOn()` 和 `observeOn()` 这两个操作符的技术和差别，原因有二：

1) 我认为搞清楚它俩是如何实现的，有助于我们消除对它俩的疑惑，因为我就是这样理解它俩的（而我再也没有疑惑过）。

2) 搞清楚它们的结构和具体行为，有助于我们理解后续进行熔合时需要的改变。

如果我们想要体验一下熔合后的效果，可以看看 [Project Reactor 2.5](https://github.com/reactor/reactor-core)，它对我在上文中描述的方案进行了充分的（单元）测试。当然，作为一个继续研究的话题，[Rsc 项目](https://github.com/reactor/reactive-streams-commons) 也欢迎大家的反馈，以及关于我们应该优化哪些操作符组合的建议。
