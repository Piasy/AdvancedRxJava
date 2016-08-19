---
layout: post
title: 调度器 Scheduler（一）：实现自定义 Scheduler
tags:
    - Scheduler
---

原文 [Schedulers (part 1)](http://akarnokd.blogspot.com/2015/05/schedulers-part-1.html){:target="_blank"}

## 介绍

`Scheduler` 是 RxJava 异步和并行计算的关键。尽管已经存在了很多标准的 scheduler，并且你也可以把一个 `Executor` 包装成一个 scheduler，我们还是应该理解 scheduler 是如何一步一步实现的，以便我们利用其他的并发资源，例如 GUI 框架的消息循环机制。

## Scheduler API

如果你对 Rx.NET 的 `IScheduler` API 比较熟悉的话，你就会发现 RxJava 的 scheduler API 稍有不同。它们的不同主要体现在处理递归调度的问题上（recursive scheduling problem）。Rx.NET 的解决方法是把实际的 scheduler 注入到被调度的任务中。

RxJava 则是借鉴了 `Iterable`/`Iterator` 模式的思想，定义了一套 `Scheduler`/`Worker` API。RxJava 的 scheduler 不进行任何调度的工作，但它负责创建 `Worker`，worker 负责实际调度，无论是直接调度还是递归调度。此外 `Worker` 还实现了 `Subscription` 接口，所以它可以被取消订阅，这会取消所有还未执行的任务，此后也不会再接受新的任务（尽可能）。这对于操作符（例如重复任务）使用 scheduler 非常有用，如果下游取消订阅了整个链条，就能一次取消所有定时的任务。

`Scheduler`/`Worker` 需要满足以下的要求：

1. 所有的方法都需要是线程安全的；
2. `Worker` 需要保证即时、串行提交的任务按照先进先出（FIFO）的顺序被执行；
3. `Worker` 需要尽可能保证被取消订阅时要取消还未执行的任务；
4. 取消订阅一个 `Worker` 不能影响同一个 Scheduler 的其他 Worker；

这些要求看起来比较严格，但这让我们对并发数据流的推算更加容易，这和我们严格要求 Observer 的方法被串行调用是一样的（_译者注：这一点不清楚的朋友可以看一下[本系列的第一篇文章的介绍部分](/AdvancedRxJava/2016/05/06/operator-concurrency-primitives/#section){:target="_blank"}_）。

除了上面必须的要求，下面几点特性如果能具备也是非常好的：

1. 一个被 Worker 调度的任务最好不要切换线程执行（hopping threads），保证在一个任务只在一个线程内执行能提升性能（避免线程切换的开销）。
2. 串行发起的延迟任务，如果延迟时间相同，最好也能按照 FIFO 的顺序执行，并发调度的任务不做此要求。

考虑到上面的这些要求，一个保守的 scheduler 实现最好用单线程的线程池来支持每个 worker，而这也正是标准 scheduler 的实现方案：底层的 `ScheduledExecutorService` 保证了上面的特性。

## 实现一个自定义的 scheduler

假设我们需要实现的 scheduler 具备以下属性：（1）只能有一个 worker 线程；（2）一个线程局部的上下文信息要能够在不同的 worker 之间传递，且能被正在执行的任务访问到。

显然，如果我们只有第一个要求，那我们可以直接利用 `Schedulers.from()` 包装一个单线程的 `Executor`，但第二个要求需要我们在一个任务被调度和执行的时候做一些额外的操作。

为了完成我们的需求，我会复用一些 RxJava 自己的 scheduler 原语：`ScheduledAction` 和 `NewThreadWorker`。（_注意，这些内部类都是随时可能发生变动的，这里我使用它们是为了避免考虑它们负责的一些细节，让我可以聚焦于我们创建 scheduler 的逻辑_）

我们先看一下类的结构：

~~~ java
public final class ContextAwareScheduler 
extends Scheduler {
    
    public static final ContextAwareScheduler INSTANCE = 
            new ContextAwareScheduler();                       // (1)
    
    final NewThreadWorker worker;
    
    private ContextAwareScheduler() {
        this.worker = new NewThreadWorker(
                new RxThreadFactory("ContextAwareScheduler")); // (2)
    }
    @Override
    public Worker createWorker() {
        return new ContextAwareWorker(worker);                 // (3)
    }
    
    static final class ContextAwareWorker extends Worker {

        final CompositeSubscription tracking;                  // (4)
        final NewThreadWorker worker;

        public ContextAwareWorker(NewThreadWorker worker) {
            this.worker = worker;
            this.tracking = new CompositeSubscription();
        }
        @Override
        public Subscription schedule(Action0 action) {
            // implement
        }
        @Override
        public Subscription schedule(Action0 action, 
                long delayTime, TimeUnit unit) {
            // implement
        }
        @Override
        public boolean isUnsubscribed() {
            return tracking.isUnsubscribed();                  // (5)
        }
        @Override
        public void unsubscribe() {
            tracking.unsubscribe();
        }
    }
}
~~~

我们的 `ContextAwareScheduler` 看起来可能有点吓人，但别怕，它的逻辑还是很直观的：

1. 由于我们只允许全局的单一线程，所以我们的 scheduler 不能存在多个实例，因此我们使用了一个静态的单例。
2. 我们的 scheduler 会把几乎所有的工作都转交给内部的一个 worker。我们复用 `NewThreadWorker` 和 `RxThreadFactory` 类来为我们的 worker 实例提供一个单一的后台线程。
3. 为了满足我们的第四个必须要求，我们不能对外也只提供一个 worker 实例，否则一旦 worker 被取消订阅，所有人的 worker 都被取消订阅了。
4. 为了满足第三个必须要求，我们需要单独为每个 worker 实例记录提交过来的任务。
5. 记录这些任务让我们可以检查是否已经取消订阅，以及进行取消订阅操作。

接下来，我们就需要前面提到的线程局部上下文了：

~~~ java
public final class ContextManager {
    static final ThreadLocal<Object> ctx = new ThreadLocal<>();
    
    private ContextManager() {
        throw new IllegalStateException();
    }
    
    public static Object get() {
        return ctx.get();
    }
    public static void set(Object context) {
        ctx.set(context);
    }
}
~~~

`ContextManager` 仅仅是包装了一个静态的 `ThreadLocal` 变量。在实际情况中，你可能要把 `Object` 替换成你需要的类型。

现在让我们继续看 `schedule()` 的实现：

~~~ java
// ...
@Override
public Subscription schedule(Action0 action) {
    return schedule(action, 0, null);               // (1)
}
@Override
public Subscription schedule(Action0 action, 
        long delayTime, TimeUnit unit) {

    if (isUnsubscribed()) {                         // (2)
        return Subscriptions.unsubscribed();
    }
    
    Object context = ContextManager.get();          // (3)
    Action0 a = () -> {
        ContextManager.set(context);                // (4)
        action.call();
    };
    
    return worker.scheduleActual(a, 
        delayTime, unit, tracking);                 // (5)
}
// ...
~~~

非常简洁！

1. 我们把即时的调度作为延迟为 0 的延迟调度。所有底层的实现都会正确解读这个 0 延迟，并且实现要求的 FIFO 顺序。
2. 如果当前的 worker 已经被取消订阅，那我们无需做任何事情，我们直接返回一个代表已经取消订阅的常量。注意，取消订阅总是需要耗费一点时间的，那就可能存在一个竞争窗口，在这个时间窗口内，可能会有一些任务（或者事件）仍然能被调度。这一竞争的处理是在 `scheduleActual` 函数中，我们后面会展开。
3. 我们获取当前线程的线程局部上下文，并把任务包装到一个新的任务中。
4. 在新的任务被执行时，我们把之前获取的局部线程上下文保存到新任务被执行的线程中。由于我们的 worker 底层都是同一个线程，所以在一个任务正在执行期间，它的线程局部上下文不会被后面的任务覆盖。
5. 最后，我们把包装好的新任务以及延迟信息委托到底层的 `NewThreadWorker` 中。由于我们把 `tracking` 传了进去，所以当任务执行完毕后，任务会被正确地移除，被取消订阅，或者和 worker 一起被取消订阅。

正如我们在上面的解释中提到的，第二步天生和 worker 的取消订阅存在竞争，但我们不能把竞争窗口内的任务丢下不管（我们需要接受它，让它被执行，或者立即取消它）。这里我们 subscription 容器类的取消订阅保证就发挥作用了（_译者注：这一点不清楚的朋友可以看一下 [subscription 容器类的第一篇文章的介绍部分](/AdvancedRxJava/2016/07/15/operator-concurrency-primitives-subscription-containers-1/#section){:target="_blank"}_）。如果我们把底层线程池返回的 `Future` 对象包装为一个 `Subscription`，那我们就可以安全地把它加到 `tracking` 容器中，它会被正确保存或者被立即取消订阅。

让我们尝试一下：

~~~ java
Worker w = INSTANCE.createWorker();

CountDownLatch cdl = new CountDownLatch(1);

ContextManager.set(1);
w.schedule(() -> {
    System.out.println(Thread.currentThread());
    System.out.println(ContextManager.get());
});

ContextManager.set(2);
w.schedule(() -> {
    System.out.println(Thread.currentThread());
    System.out.println(ContextManager.get());
    cdl.countDown();
});

cdl.await();

ContextManager.set(3);

Observable.timer(500, TimeUnit.MILLISECONDS, INSTANCE)
.doOnNext(v -> {
    System.out.println(Thread.currentThread());
    System.out.println(ContextManager.get());
}).toBlocking().first();

w.unsubscribe();
~~~

_译者注，运行输出如下：_

~~~ bash
Thread[ContextAwareScheduler1,5,main]
1
Thread[ContextAwareScheduler1,5,main]
2
Thread[ContextAwareScheduler1,5,main]
3
~~~

## 总结

Scheduler API 让我们有机会指定与 Observable 链条操作相关的任务在何地、甚至何时被执行。内置的标准 scheduler 应该能满足大部分并发的需求，但还是存在一些场景，我们需要使用和实现自定义的 scheduler。在本文中，我展示了如何利用 RxJava 已有的一些类来实现具有自定义逻辑的自定义 scheduler。

在接下来的文章中，我们会深入介绍 `ScheduledAction` 的原理，并且借助它的一些概念，让我们在调度任务时进行更多的控制，例如和线程中断协调工作。
