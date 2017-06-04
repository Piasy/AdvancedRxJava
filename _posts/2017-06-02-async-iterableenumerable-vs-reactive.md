---
layout: post
title: Async Iterable/Enumerable vs. Reactive-Streams
tags:
    - 对比点评
---

原文 [Async Iterable/Enumerable vs. Reactive-Streams](http://akarnokd.blogspot.com/2016/05/async-iterableenumerable-vs-reactive.html){:target="_blank"}。

## 介绍

在响应式数据处理流水线中，当两个部分处理数据速度不同时，如果我们不希望缓冲区爆炸或者内存耗尽，那 backpressure 就至关重要了。RxJava 和 Reactive-Streams 设计了一套非阻塞、基于请求协调的处理机制，但可能我们也听说过其他的解决方案。一个经常被提及的概念就是 Async Iterables（Java 中的概念）或者 Async Enumerables（C# 中的概念）了。

实际上 Rx.NET 有一个子项目 Ix.NET（Interactive Extensions 的缩写），其中就使用了 [Async Enumerables](https://github.com/Reactive-Extensions/Rx.NET/tree/2252cb4edbb25aca12005b9a912311edd2f095f3/Ix.NET/Source/System.Interactive.Async)。它通过 `MoveNext()`（相当于 `hasNext()`）返回一个 `Task`（相当于 `CompletableFuture`、`Promise`）来解决 backpressure 的问题：当 `Task` 完成时，我们可以消费 `Current` 的值（相当于 `next()` 函数），由于我们只能在消费了当前的数据之后才能调用 `MoveNext()`，所以自然就解决了 backpressure 的问题。

遗憾的是，我没有找到 `IAsyncEnumerable` 的 Java 实现（不过我也只是用 Google 搜索了几次），所以我决定用 Java 8 自己实现一下，看看让数据在其中流动需要做些什么，以及对比一下它和我当前对响应式数据流的最新理解—— [Reactive-Streams-Commons](https://github.com/reactor/reactive-streams-commons) ——的性能差异。

## 基本 API

由于 Async Enumerables 的设计从源头上考虑了延迟执行，所以基本 API 包括下面两个接口：

~~~ java
interface IAsyncEnumerable<T> {
    IAsyncEnumerator<T> enumerator();
}
 
interface IAsyncEnumerator<T> {
 
    CompletionStage<Boolean> moveNext(CompositeSubscription cancel);
 
    T current();
}
~~~

`IAsyncEnumerable` 相当于是 `Iterable`，它能发出 `IAsyncEnumerator`。`IAsyncEnumerator` 的 `moveNext` 函数能返回一个 `CompletionStage`，它能用于表明当前能否通过 `current()` 获取数据（`true` 表示能，`false` 表示数据流终止）。C# 的 `CancellationToken` 和 `CompositeSubscription` 类似，所以我就用它来实现取消了。

（注：我现在还没想好怎么实现组合取消，Ix.NET 的 `IAsyncEnumerator` 是一个 `IDisposable`，而且 `Task` 也可以取消，不像 `CompletionStage` 无法取消。不过幸运的是，在这篇博文中，我并不太需要这个特性。）

## 消费 IAsyncEnumerable

消费一个 `IAsyncEnumerable` 还是很直观的，尽管不像 C# 的 `async`/`await` 那样简单。如果我们只关心单一数据，那我们可以这样实现：

~~~ java
IAsyncEnumerable<T> source = ...
 
IAsyncEnumerator<T> enumerator = source.enumerator();
 
enumerator.moveNext(new CompositeSubscription())
.whenComplete((b, e) -> {
    if (e != null) {
        e.printStackTrace();
    } else if (b) {
        System.out.println(enumerator.current());
    } else {
        System.out.println("Empty!");
    }
});
~~~

当然，基于 `CompletionStage` 的 API，我们可以这样轻易地处理单一的数据。

但如果要消费 `IAsyncEnumerator` 中的多个数据，那就要做更多事情了，我们需要递归调用 `moveNext`，直到数据流终止，或者出现错误：

~~~ java
public void consumeAll(IAsyncEnumerator<T> enumerator, CompositeSubscription csub) {
    if (csub == null) {
        csub = new CompositeSubscription();
    }
 
    CompositeSubscription fcsub = csub;
 
    enumerator.moveNext(new CompositeSubscription())
    .whenComplete((b, e) -> {
        if (e != null) {
            e.printStackTrace();
        } else if (b) {
            System.out.println(enumerator.current());
 
            // go recursive            
            consumeAll(enumerator, fcsub);
        } else {
            System.out.println("Empty!");
        }
    });    
}
~~~

不幸的是，如果 `CompletionStage` 是同步的，那我们可能遇到 `StackOverflowError`，因为我们在无限调用 `consumeAll`。因此我们需要一个线程跳转，以确保调用栈不会太深：

~~~ java
public final class AsyncConsumer<T> implements Subscription {
 
    final Consumer<? super T> onNext;
 
    final Consumer<Throwable> onError;
 
    final Runnable onComplete;
 
    final IAsyncEnumerator<T> enumerator;
 
    final AtomicInteger wip;
 
    final Queue<CompletionStage<Boolean>> queue;
 
    final CompositeSubscription csub;
 
    final CountDownLatch cdl;
 
    public AsyncConsumer(
         IAsyncEnumerator<T> enumerator,
         Consumer<? super T> onNext,
         Consumer<Throwable> onError,
         Runnable onComplete
    ) {
         this.enumerator = enumerator;
         this.onNext = onNext;
         this.onError = onError;
         this.onComplete = onComplete;
         this.wip = new AtomicInteger();
         this.queue = new SpscLinkedArrayQueue<>(16);
         this.csub = new CompositeSubscription();
         this.cdl = new CountDownLatch();
    }
 
    public void consumeAll() {
         if (csub.isUnsubscribed()) {
             cdl.countDown();
             return;
         }
         CompletionStage<T> stage = enumerator.moveNext(csub);
         queue.offer(stage);
         if (wip.getAndIncrement() == 0) {
             do {
                 stage = queue.poll();
                 stage.whenComplete((b, e) -> {
                     if (csub.isUnsubscribed()) {
                         cdl.countDown();
                         return;
                     } else if (e != null) {
                         onError.accept(e);
                         cdl.countDown();
                     } else if (b) {
                         onNext.accept(enumerator.current());
                         consumeAll();
                     } else {
                         onComplete.run();
                         cdl.countDown();
                     }
                 });
             } while (wip.decrementAndGet() != 0);
         }
    }
 
    public void await() throws InterruptedException {
        cdl.await();
    }
 
    public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
        return cdl.await(timeout, unit);
    }
 
    @Override
    public void unsubscribe() {
         csub.unsubscribe();
    }
 
    @Override
    public boolean isUnsubscribed() {
         return csub.isUnsubscribed();
    }
}
~~~

看起来很吸引人，除了发出事件的回调以及 `IAsyncEnumerator`，我们还需要一个 `wip` 计数器和 `queue`，就像以前的队列漏一样。我们用 `CompositeSubscription` 实现取消，用 `CountDownLatch` 实现阻塞等待。`consumeAll()` 每次消费一个数据，线程跳转的循环确保了每次只会有一个 `whenComplete()` 处于运行中。在回调中，我们用 `Consumer` 发出相应的事件，如果来了数据，那就递归调用 `consumeAll`。线程的跳转确保了无论是同步调用还是异步调用，我们都不会重入 `consumeAll`。

## 编写 IAsyncEnumerable 数据源

当然我们需要数据源才能工作，作为所有响应式编程库中最基础、最标准的数据源——`range()`——可以说是响应式编程领域的 for 循环了。

~~~ java
public final class AsyncRange implements IAsyncEnumerable<Integer> {
    final int start;
    final int count;
 
    @Override
    public IAsyncEnumerator<Integer> enumerator() {
        return new AsyncRangeEnumerator(start, count);
    }
 
    static final class AsyncRangeEnumerator implements IAsyncEnumerator<Integer> {
        final long end;
        long index;
 
        static final CompletionStage<Boolean> TRUE = 
            CompletableFuture.completedFuture(true);
 
        static final CompletionStage<Boolean> FALSE = 
            CompletableFuture.completedFuture(false);
 
        public AsyncRangeEnumerator(int start, int count) {
            this.index = start - 1;
            this.end = (long)start + count;
        }
 
        @Override
        public CompletionStage<Boolean> moveNext(CompositeSubscription csub) {
            long i = index + 1;
            if (i == end) {
                return FALSE;
            }
            index = i;
            return TRUE;
        }
 
        @Override
        public Integer current() {
            return index;
        }
    }
}
~~~

range 操作本身是同步的，在我看来，`CompletableFuture` 和 `AsyncSubject` 一样，而由于我们需要返回的是 `true` 或者 `false` 常量，因此我们可以使用共享的、已结束的 Future 对象。由于 `moveNext()` 会在 `current()` 之前调用，所以 `index` 初始值是 `start - 1`，并在 `moveNext()` 中把 `index` 加一。如果 `index == end`，那我们就返回 `false`，否则我们把 `index` 加一然后返回 `true`。

## 异步化

当然我们需要的是异步，所以让我们实现一下 `observeOn` 和 `subscribeOn` 操作符。前者确保 `CompletionStage<Boolean>` 的代码在指定的线程执行，后者确保 `moveNext()` 的代码执行在指定的线程（所以我们就可以在 `moveNext()` 或者 `enumerator()` 中进行阻塞 IO 操作了）。此外，[编写这两个操作符](/AdvancedRxJava/2016/09/16/subscribeon-and-observeon/)我们已经很熟练了，对吧？

### observeOn

如果大家有些不熟悉的操作符要实现，那我建议看看 Erik Meijer 在 Channel 9 的视频：follow the types。我们知道，我们有一个上游数据源，以及一些异步数据源。为了简洁起见，我们直接使用 `Executor`，因为 `CompletionStage` 的 `XXXAsync` 函数和 `Executor` 几乎一模一样。

~~~ java
public final class AsyncObserveOn<T> implements IAsyncEnumerable<T> {
    final IAsyncEnumerable<T> source;
 
    final Executor executor;
 
    public AsyncObserveOn(IAsyncEnumerable<T> source, Executor executor) {
        this.source = source;
        this.executor = executor;
    }
 
    @Override
    public IAsyncEnumerator<T> enumerator() {
        return new AsyncObserveOnEnumerator<>(source.enumerator(), executor);
    }
 
    // ... 
}
~~~

这个套路大家应该都很熟悉了，脏活累活都被 `AsyncObserveOnEnumerator` 包揽了：

~~~ java
static final class AsyncObserveOnEnumerator<T> implements IAsyncEnumerator<T> {
 
    final IAsyncEnumerator<T> enumerator;
 
    final Executor executor;
 
    public AsyncObserveOnEnumerator(IAsyncEnumerator<T> enumerator, Executor executor) {
        this.enumerator = enumerator;
        this.executor = executor;
    }
     
    @Override
    public CompletionStage<Boolean> moveNext(CompositeSubscription csub) {
        return enumerator.moveNext(csub).thenApplyAsync(v -> v, executor);
    }
 
    @Override
    public T current() {
        return enumerator.current();
    }
}
~~~

我承认我对 `CompletionStage` 不是很熟悉，所以我觉得 `thenApplyAsync` 是把数据送到指定 `Executor` 最简单的办法了。其他的代码看起来都很直观了，而且比咱们 Rx 风格的 `observeOn()` 简短得多。

### subscribeOn

由于我们不知道 `IAsyncEnumerable` 是否会在 `moveNext()` 中实现异步，我们必须想个办法（`subscribeOn`），确保 `moveNext()` 发生在另外的线程：

~~~ java
public final class AsyncSubscribeOn<T> implements IAsyncEnumerable<T> {
    final IAsyncEnumerable<T> source;
 
    final Executor executor;
 
    public AsyncSubscribeOn(IAsyncEnumerable<T> source, Executor executor) {
        this.source = source;
        this.executor = executor;
    }
 
    @Override
    public IAsyncEnumerator<T> enumerator() {
        AxSubscribeOnEnumerator<t> enumerator = new AxSubscribeOnEnumerator<>(executor);
        executor.execute(() -> {
            IAsyncEnumerator<T> ae = source.enumerator();
            enumerator.setEnumerator(ae);
        });
        return enumerator;
    }
 
    // ... 
}
~~~

我们把 `source.enumerator()` 的调用转移到了 `Executor` 中，而不是直接调用，在调用 `enumerator()` 时我们将会有一个合法的 `IAsyncEnumerator`（这一操作在消费者看来被推迟了），不过我们仍需要返回一个 `IAsyncEnumerator`。这里的困难之处就在于如何允许在没有合法的上游 `Enumerator` 时调用 `moveNext()`。幸好，`CompletionStage` 能帮我们解决这一难题：

~~~ java
static final class AxSubscribeOnEnumerator<T> implements IAsyncEnumerator<T> {
 
    final Executor executor;
     
    final CompletableFuture<IAsyncEnumerator<T>> onEnumerator;
     
    public AxSubscribeOnEnumerator(Executor executor) {
        this.executor = executor;
        this.onEnumerator = new CompletableFuture<>();
    }
     
    void setEnumerator(IAsyncEnumerator<T> enumerator) {
        onEnumerator.complete(enumerator); 
    }
     
    @Override
    public CompletionStage<Boolean> moveNext(CompositeSubscription token) {
        return onEnumerator.thenComposeAsync(ae -> ae.moveNext(token), executor);
    }
 
    @Override
    public T current() {
        IAsyncEnumerator<T> ae = onEnumerator.getNow(null);
        return ae != null ? ae.current() : null;
    }
     
}
~~~

我们准备了一个 `onEnumerator` `CompletableFuture`，当 `setEnumerator` 调用时，我们将获得实际的上游 `IAsyncEnumerator`，这时 Future 将会完成。这里最大的技巧在于，当 `onEnumerator` 的值到达时，我们立即在 `Executor` 上调用上游的 `moveNext()` 方法。`thenComposeAsync` 操作符和 `flatMap` 差不多。`current()` 函数则需要一些额外的逻辑，我们先尝试获取上游 `Enumerator`，如果此时还没有，那就直接返回 `null`，如果相应的 `CompletionStage` 还没有发出之前，我们不能调用 `current()`。

## 基准测试

既然我们有了这三个最基础的操作符，那就为 `IAsyncEnumerable` 和最新的 Reactive-Streams-Commons 做一个基准测试的对比吧。相关的源码可以从 [GitHub 获取](https://github.com/akarnokd/akarnokd-misc/blob/master/src/jmh/java/hu/akarnokd/asyncenum/IAsyncEnumerablePerf.java)。为了方便，我把上述操作符实现为了流式 API，基本类型是 `Ax`：Async Extensions。

下面是基准测试的结果（数值越大越好）：（i7 4790, Windows 7 x64, Java 8u92）

![](https://imgs.piasy.com/2017-06-04-ax_benchmark.png)

`range` 测试的是 `range(1, count)`，`rangeAsync` 测试的是 `range(1, count).observeOn(executor)`，`rangePipeline` 测试的是 `range(1, count).subscribeOn(executor1).observeOn(executor2)`。`ax` 代表我实现的 `IAsyncEnumerable`，`px` 代表 Reactive-Streams-Commons (Rsc) 的 Publisher Extensions。由于 Rsc 实现了操作符熔合，因此 `rangeAsync()` 在启用和禁用操作符熔合的情况下分别作了测试，`pxf` 代表启用了操作符熔合的 Rsc。

## 评估

看起来 Rsc 完爆了我实现的 `IAsyncEnumerable`，无论是同步模式还是异步模式。由于没有第三方的实现可以做对比，我只能猜测 `IAsyncEnumerable` 开销如此之大的原因了。我有限的 `CompletionStage` 使用经验告诉我一些原因，但我怀疑这并非根源所在。由于两个库都是使用了单线程 `Executor`，所以可以排除 `Executor` 的开销。

剩下的就是架构上、概念上的差异了：

+ 每个数据都需要创建一个 `CompletionStage`，以及一个消费者；Rsc 没有任何内存分配；
+ `CompletionStage` 的行为介于 hot 和 cold 之间，就像 `AsyncSubject` 那样，当我们把消费者连接上去的时候，它可能仍在运行，也可能早已停止；这项检查增加了额外开销；Rsc 则是尽可能直接调用 `onNext`；
+ 调用链越长，就需要越多的 `CompletionStage`，这就意味着内存分配和单独的任务调度；Rsc 则在异步边界上实现了批处理机制；

## 总结

我觉得这里的 `IAsyncEnumerable` 就是响应式数据流的一个特例，它每次都调用 `request(1)`，也存在一些内存分配的开销，使得和高度优化的响应式数据流相比，开销高得多。

当然它看起来更精简，操作符的实现也更简单，但我不得不问，它和 Reactive-Streams 相比，有何优势？

如果大家对优化 `IAsyncEnumerable` 有任何建议，或者能推荐第三方的实现，我将很乐意重新做一次基准测试，并调整我对这个话题的观点。
