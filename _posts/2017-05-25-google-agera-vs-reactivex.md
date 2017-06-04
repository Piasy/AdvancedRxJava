---
layout: post
title: Google Agera vs. ReactiveX
tags:
    - 对比点评
---

原文 [Google Agera vs. ReactiveX](http://akarnokd.blogspot.com/2016/04/google-agera-vs-reactivex.html){:target="_blank"}。

## 介绍

如果大家一直关注安卓开发圈内的事件，或者关注响应式编程相关的事件，那就知道最近 Google 搞了一件“大事情”：他们发布了专门针对安卓平台的响应式编程库 [Agera](https://github.com/google/agera)。（_译者注：其实 Agera 是一年前的事了，不过一年过去了，Agera 好像没什么动静了_）当然，我们需要仔细深入细节，才能准确了解 Agera 是怎么回事。

“Google 制造”的意思其实是一个负责 Google Play 电影 APP 的团队制造。当然，说 Google 显然比说出这个团队的完整名字要震撼得多。当别人问我在哪里工作时我也会采取同样的策略：我会说在 **lab at the Hungarian Academy of Sciences** 而不是 **the Engineering and Management Intelligence Research Laboratory at the Institute for Computer Science and Control of the Hungarian Academy of Sciences**。（_不要在意原作者到底在哪里工作，都是浮云..._）

谁发布的并不重要，重要的是发布了啥，以及它和现有的响应式编程库是啥关系：[RxJava](https://github.com/ReactiveX/RxJava)，[Reactor](https://github.com/reactor/reactor-core)，[Akka-Streams](https://github.com/akka/akka/tree/master/akka-stream)。

## 核心 API

Agera 库的基本思想是观察者模式：`Observable` 接收 `Updatable`，并通过 `update()` 接口通知新事件。之后搞清楚更新的内容就是 `Updatable` 自己的事了。这毫无疑问是一种依赖于 `update()` 调用副作用的响应式数据流。

~~~ java
interface Updatable {
    void update();
}
 
interface Observable {
   void addUpdatable(Updatable u);
   void removeUpdatable(Updatable u);
}
~~~

它们看起来人畜无害，而且很响应式对吧？然而不幸的是，它们有和 `java.util.Observable` 以及基于 `addListener`/`removeListener` 的 API 一样的问题，这种 API 我在[前文中归为第 0 代响应式编程库](/AdvancedRxJava/2017/05/01/operator-fusion-part-1/#section-2)，可以看看[我在 GitHub 上的 issue 评论](https://github.com/google/agera/issues/20#issuecomment-212007539)。

### Agera Observable

这样一对接口的问题在于，每个想要通知 `Updatable` 后续更新的 `Observable`，都需要保存 `Updatable` 对象的引用，以便后续移除之：

~~~ java
public final class DoOnUpdate implements Observable {
    final Observable source;
 
    final Runnable action;
 
    final ConcurrentHashMap<Updatable, DoOnUpdatable> map;
 
    public DoOnUpdate(Observable source, Runnable action) {
         this.source = source;
         this.action = action;
         this.map = new ConcurrentHashMap<>();
    }
 
    @Override
    public void addUpdatable(Updatable u) {
        DoOnUpdatable wrapper = new DoOnUpdatable(u, action);
        if (map.putIfAbsent(u, wrapper) != null) {
            throw new IllegalStateException("Updatable already registered");
        }
        source.addUpdatable(wrapper);
    }
 
    public void removeUpdatable(Updatable u) {
        DoOnUpdatable wrapper = map.remove(u);
        if (wrapper == null) {
            throw new IllegalStateException("Updatable already removed");
        }
        source.removeUpdatable(wrapper);
    }
 
    static final class DoOnUpdatable {
        final Updatable actual;
 
        final Runnable run;
 
        public DoOnUpdatable(Updatable actual, Runnable run) {
            this.actual = actual;
            this.run = run;
        }
 
        @Override
        public void update() {
            run.run();
            actual.update();
        }
    }
}
~~~

而这导致了毫不相关的下游 `Updatable` 之间，在每个环节都存在竞争点。

的确，RxJava 的 [Subject](/AdvancedRxJava/2016/10/05/subjects-part-3/) 和 [ConnectableObservable](/AdvancedRxJava/2017/03/03/connectableobservables-part-2/) 中也存在类似的竞争点，但它们之后链起来的操作符之间是不存在竞争的。不幸的是，Reactive-Streams 规范当前版本是禁止 `Publisher` 存在类似的竞争的，但 RxJava，Rsc 和 Reactor 都忽略了这一点，这其实是过于保守了，我们也正在努力矫枉，以让规范更轻量。

第二个问题没这么严重，那就是我们没法添加同一个 `Updatable` 多次。首先因为我们用 `Map` 无法区分不同的“订阅关系”，其次，Agera 规范也要求在这种情况下抛出异常。当然，通常这种情况都不会发生，因为大多数末端消费者都只用一次。

第三个问题则严重一些了：当 `Updatable` 没有注册到 `Observable` 上时，抛出异常。这就导致了末端消费者和中间操作符移除 `Updatable` 的竞争，而且它俩之间必有一者会抛出异常。这正是现代响应式编程库的取消操作都是幂等的原因。

第四个问题则是，`addUpdatable` 和 `removeUpdatable` 之间理论上会存在竞争：下游操作符可能希望在上游调用 `addUpdatable` 之前就断开连接。这导致的结果就是 `removeUpdate` 抛出异常，但 `addUpdatable` 成功，使得数据流无论如何都会继续，而且会造成不必要的对象引用。

### Agera Updatable

让我们看看消费者这边的 API。`Updatable` 是个函数式接口，这样我们为 `Observable` 增加一个监听者就比较简洁：

~~~ java
Observable source = ...
 
source.addUpdatable(() -> System.out.println("Something happened"));
~~~

足够简单，现在让我们移除一个监听者：

~~~ java
source.removeUpdatable(() -> System.out.println("Something happened"));
~~~

但这会抛出异常，因为这两个 lambda 表达式不是同一个实例。这是基于 `addListener`/`removeListener` API 的常见问题，解决办法就是把 lambda 表达式存起来再用：

~~~ java
Updatable u = () -> System.out.println("Something happened");
 
source.addUpdatable(u);
 
// ...
 
source.removeUpdatable(u);
~~~

确实只是一个小小的不便，但情况会恶化。如果我们有很多个 `Observable` 和 `Updatable` 怎么办？那我们就不得不记住谁注册到了谁上面，而且用变量保存起来。最初 Rx.NET 一个很好的点子就是通过一个接口来省去所有的这些不便：

~~~ java
interface Removable extends Closeable {
    @Override
    void close(); // remove the necessity of try-catch around close()
}
 
public static Removable registerWith(Observable source, Updatable consumer) {
    source.addUpdatable(consumer);
    return () -> source.removeUpdatable(consumer);
}
~~~

当然，这里我们也要考虑到 `close()` 的幂等性：

~~~ java
public static Removable registerWith(Observable source, Updatable consumer) {
    source.addUpdatable(consumer);
    final AtomicBoolean once = new AtomicBoolean();
    return () -> {
        if (once.compareAndSet(false, true)) {
            source.removeUpdatable(consumer);
        }
    });
}
~~~

### Agera MutableRepository

`MutableRepository` 存有一个可变值，并且当这个值发生改变时调用 `update()` 通知已注册的 `Updatable`。这在某种程度上模拟了 `BehaviorSubject`，区别是新值不会传给下游（因为 `update()` 没有参数），下游需要通过 `get()` 主动获取：

~~~ java
MutableRepository<Integer> repo = Repositories.mutableRepository(0);
 
repo.addUpdatable(() -> System.out.println("Value: " + repo.get());
 
new Thread(() -> {
    repo.accept(1);
}).start();
~~~

当通过工厂方法创建 `MutableRepository` 时，它就具备了一个有趣的特性：`update()` 将在创建 `MutableRepository` 的线程被调用。（`Looper` 就像是每个线程私有的跳板调度器/ `Executor`，使得我们可以在指定的线程上执行代码，例如安卓的主线程）

这一特性就导致了下面这个有趣的情况：

~~~ java
Set<Integer> set = new HashSet<>();
 
MutableRepository<integer> repo = Repositories.mutableRepository(0);
 
repo.addUpdatable(() -> set.add(repo.get()));
 
new Thread(() -> {
    for (int i = 0; i < 100_000; i++) {
        repo.accept(i);
    }
}).start();
 
Thread.sleep(20_000);
 
System.out.println(set.size());
~~~

[假设 20s 足够了](https://github.com/google/agera/issues/31)，猜猜 `Set` 打印出来的大小是多少？我们可能会预期 `Set` 里面有所有 100000 个整数，但实际上可能是 1~100000 之间的任何数字！原因是 `accept()` 和 `get()` 并发执行，如果消费者速度慢，那 `accept()` 就会覆盖 `MutableRepository` 中的值。

有些情况下，这是可以接收的（就像应用了 RxJava 的 `onBackpressureDrop` 运算符一样），但有些情况下时不可接受的，因此我们可能会花费大量的时间排查为何数据会丢失。

### 错误处理

异步化通常意味着错误也得异步处理。RxJava 以及其他的响应式编程库在这一点上处理得很好：一旦某个环节发生了错误，整个处理流程就自动清理干净了，除非程序员希望错误发生后忽略错误、替代为默认值或者重试。错误处理和清理机制有时非常复杂，但库的开发者为此付出了巨大的努力，因此大家大多数情况下都不需要考虑这个问题。

Agera 的基本 API 并没有考虑到错误处理，我们必须自己处理错误，就像处理数据一样。如果我们有多个 Agera 服务组合在一起，那我们必须实现一套在 callback-hell 情况下需要实现的错误处理机制。由于并发和终止状态的考虑，实现这样的错误处理机制是非常麻烦而且容易出错的。

### 终止状态

Agera 没有对已终止数据流的表述方式：我们必须自行确定终止时间。这在 GUI 系统中常常没有问题，因为用户始终都在和界面打交道，不会终止。但是后台的异步任务就必须通过某种方式通知自己会发出多少事件，以及从何时开始 `update()` 将不会被调用了。

## 如何设计一套现代的无参数响应式 API

首先，我们不应该考虑自行设计，而是利用已有的接口设计：

~~~ java
rx.Observable<Void> signaller = ...
 
rx.Observer<Void> consumer = ...
 
Subscription s = signaller.subscribe(consumer);
 
// ...
 
s.unsubscribe();
~~~

我们立即就拥有了所有的基础设施，操作符，以及高性能，几乎不需要付出任何代价。而且如果我们希望处理有意义的数据更新，那我们可以把 `Void` 替换为相应的数据类型。

如果现有的库看起来太重了，拥有太多用不到的操作符，那我们可以 fork 一份，删掉用不着的内容，然后利用之。当然，接下来我们需要保持更新，以应用 bugfix 和性能优化。

如果 fork 和删代码看起来不吸引人，那我们完全可以在 Reactive-Streams 规范的基础上实现自己的响应式编程库：`Publisher<Void>` 和 `Subscriber<Void>`，以及其他我们需要的代码。我们立刻就免费获得了和其他基于 Reactive-Streams 规范的响应式编程库交互的能力，此外，我们还能利用兼容性测试套件（TCK）来测试我们的实现。

当然，编写响应式编程库很难，基于 Reactive-Streams 规范实现更难。因此，最终的故事很可能是我们会从头实现一套准系统的 API。

如果大家确实希望实现一套自己的无参数响应式数据流，下面几点建议可以考虑一下：

**1）不要有单独的 `addListener` 和 `removeListener` 接口**。单一的接口将会简化中间操作符的开发：

~~~ java
interface Observable {
    Removable register(Updatable u);
}
 
interface Removable {
    void remove();
}
~~~

**2）考虑把取消/移除的支持注入到目标位置，而不是返回一个对象**：

~~~ java
interface Observable {
    void register(Updatable u);
}
 
interface Updatable {
    void onRegister(Removable remover);
    void update();
}
 
// or
 
interface Updatable {
    void update(Removable remover);
}
~~~

**3）至少考虑一下错误的传递**：

当然，这增加了库开发者的工作量，但能极大地简化库使用者的工作。

~~~ java
interface Updatable {
    void onRegister(Removable remover);
    void update();
    void error(Throwable ex);
}
~~~

**4）提供可选的异步支持**。

在 `MutableRepository` 的例子中，我们可能希望在回到主线程之前，在调用者线程上线处理一下数据。这就意味着需要 [observeOn](/AdvancedRxJava/2016/09/16/subscribeon-and-observeon/) 或者 `subscribeOn` 了（如果我们需要处理冷数据源的话）。

## 总结

编写一个响应式编程库并非易事，如果我们不熟悉这个领域的历史和演进，那我们可能会陷进很多坑里面。在很多公司里面，“非此制造”和“我们能做得更好”等理念如此强烈，以至于他们宁愿自己从头开始写，也不愿意基于他人已有的工作。

（_有趣的是，当我向很多内部项目推荐 RxJava 时，对方仍表示很惊讶（getting raised eyebrows），即便他们已经快要自己编出一套 RxJava 了。_）

大家可能会问，我为啥要关心 Google/Agera？我对 RxJava 不自信吗？我当然很自信，而且 Agera 的出现对 RxJava 半毛钱影响都没有！

然而，根据[我以往的经验](/AdvancedRxJava/2017/03/25/asynchronous-event-streams-vs-reactive/)，如果有人空有噱头，口说无凭，盲目自信，那整个社区都可能会被带坏。我并不想对这件事做出太多评价，但想象一下，如果安卓的下一个版本强制以当前这个形态的 Agera 作为**标准的**安卓异步编程库（_那将多么糟糕..._）。

（此外，[互操作](https://github.com/akarnokd/RxAgera)有时是不可避免的，我并不希望如果它们无法很好地互操作时，在 RxJava 的 issue 里面看到大家的抱怨。）

让我以一句慧语来作为结尾（目前有两个例证了）：**你想要编写自己的响应式编程库吗？请别这样做！**
