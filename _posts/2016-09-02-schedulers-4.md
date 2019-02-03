---
layout: post
title: 调度器 Scheduler（四，完结）：实现 GUI 系统的 Scheduler
tags:
    - Scheduler
---

原文 [Schedulers (part 4 - final)](http://akarnokd.blogspot.com/2015/06/schedulers-part-4-final.html){:target="_blank"}

## 介绍

在关于 `Scheduler` 的最后一篇文章（本文）中，我将讲讲如何在那些没有暴露出 `ExecutorService` 的系统中实现 scheduler，例如 GUI 的事件循环线程。

## 不基于 `Executor` 服务的 scheduler

在一些类似于 Java AWT 事件循环的调度系统中，并没有提供 `Future` 这样的 API，只有一个提交任务的方法。另一些系统可能会提供一些成对的提交/移除任务的方法。

假设我们在一些 GUI 事件循环系统中有如下的 API 可以提交任务和取消任务：

~~~ java
public interface GuiEventLoop {
    void run(Runnable task);
    void cancel(Runnable task);
}
 
public static final class EDTEventLoop 
implements GuiEventLoop {
    @Override
    public void run(Runnable task) {
        SwingUtilities.invokeLater(task);
    }
 
    @Override
    public void cancel(Runnable task) {
        // not supported
    }
}
~~~

在这儿我定义了一个示例的 API，它的实现将会对 Java AWT 的 **事件分发线程（Event Dispatch Thread）**进行一个包装。不幸的是，EDT 不支持取消已经提交的任务，不过由于被提交的任务本来就不支持耗时操作，所以这一点对应用程序来说并不是大问题。

我们会很自然地想到直接把上面的调用包装为一个 `Executor`：

~~~ java
Executor exec = SwingUtilities::invokeLater;
~~~

然后利用[本系列第三篇](/AdvancedRxJava/2016/08/26/schedulers-3/index.html){:target="_blank"}中介绍的 `ExecutorScheduler`，但这种方式通常会带来一些额外的开销，而且我也会展示当我们可以通过 GUI 系统的某些方法取消、移除任务时，我们可以怎样处理这一问题。

由于 GUI 的事件循环是单线程的，所以我们在实现 `Worker` 时无需考虑同步和中继问题，让我们看看更简单的 `GuiScheduler` 的类结构：

~~~ java
public final class GuiScheduler extends Scheduler {
     
    final GuiEventLoop eventLoop;
     
    public GuiScheduler(GuiEventLoop el) {
        this.eventLoop = el;
    }
     
    @Override
    public Worker createWorker() {
        return new GuiWorker();
    }
 
    final class GuiWorker extends Worker {
        final CompositeSubscription tracking = 
            new CompositeSubscription();
        @Override
        public void unsubscribe() {
            tracking.unsubscribe();
        }
 
        @Override
        public boolean isUnsubscribed() {
            return tracking.isUnsubscribed();
        }
 
        @Override
        public Subscription schedule(Action0 action) {
            // implement
        }
 
        @Override
        public Subscription schedule(
                Action0 action, 
                long delayTime,
                TimeUnit unit) {
            // implement
        }
    }
}
~~~

到目前为止还没有什么特殊之处：我们把所有的调度都转发到同一个 `GuiEventLoop` 实例中，并且单独记录被提交到每个 `Worker` 上的任务。由于我们假定 `GuiEventLoop` 是单线程的，所以就不必要实现队列漏逻辑了。下面我们首先看看非延迟的 `schedule()` 实现：

~~~ java
@Override
public Subscription schedule(Action0 action) {
    if (isUnsubscribed()) {                             // (1)
        return Subscriptions.unsubscribed();
    }
    ScheduledAction sa = new ScheduledAction(action);
    tracking.add(sa);
    sa.add(Subscriptions.create(
            () -> tracking.remove(sa)));                // (2)
     
    Runnable r = () -> {                                // (3)
        if (!sa.isUnsubscribed()) {
            sa.run();
        }
    };
     
    eventLoop.run(r);                                   // (4)
     
    sa.add(Subscriptions.create(
            () -> eventLoop.cancel(r)));                // (5)
     
    return sa;
}
~~~

实现逻辑我们已经很熟悉了：

1. 如果 worker 已经被取消订阅，我们就直接返回代表已取消订阅的 `Subscription` 实例。
2. 我们把任务包装为一个 `ScheduledAction`，并且实现记录和取消记录的逻辑。
3. 在这个例子中，`ScheduledAction` 是否已经被取消有必要区分，所以如果已经取消，那我们就不执行它。由于 `GuiEventLoop` 在取消任务时需要同一个 `Runnable` 实例，并且 `ScheduledAction.run()` 中也不检查 `isUnsubscribed()`，所以我们需要把执行和取消逻辑包装到一个 `Runnable` 中。
4. 我们把 Runnable 提交到 GuiEventLoop 中。
5. 然后我们在取消订阅时移除提交的任务。注意，如果我们交换（4）和（5），而且就在我们执行 `eventLoop.run(r)` 之前 worker 被取消订阅了，那我们就会立即移除 `r`（而此时 `r` 并不在 GuiEventLoop 中），那我们再执行 `eventLoop.run(r)` 时，就无法取消了。

由于我们要适配的 API 不提供延迟执行的功能（延迟执行在处理定期执行时将很有用，例如动画），所以我们依然要利用[本系列第二篇](/AdvancedRxJava/2016/08/19/schedulers-2/index.html){:target="_blank"}中介绍的 `genericScheduler`：

~~~ java
@Override
public Subscription schedule(
        Action0 action, 
        long delayTime,
        TimeUnit unit) {
 
    if (delayTime <= 0) {                             // (1)
        return schedule(action);
    }
    if (isUnsubscribed()) {                           // (2)
        return Subscriptions.unsubscribed();
    }
    ScheduledAction sa = 
            new ScheduledAction(action);
    tracking.add(sa);
    sa.add(Subscriptions.create(
            () -> tracking.remove(sa)));              // (3)
 
    Future<?> f = CustomWorker.genericScheduler       // (4)
    .schedule(() -> {
        Runnable r = () -> {
            if (!sa.isUnsubscribed()) {
                sa.run();
            }
        };
         
        eventLoop.run(r);
         
        sa.add(Subscriptions.create(
                () -> eventLoop.cancel(r)));          // (5)
         
    }, delayTime, unit);
 
    sa.add(Subscriptions.create(
            () -> f.cancel(false)));
 
    return sa;
}
~~~

这次我们延迟调度的算法都归结到了和非延迟调度相同的处理流程上，两者非常相似：

1. 我们把非正的延迟当成是对非延迟 `schedule()` 调用。
2. 如果 worker 已经取消订阅，我们直接返回。
3. 我们实现记录和取消记录逻辑。
4. 我们利用 `genericScheduler` 来延迟在 `eventLoop` 上执行任务。
5. 在 `eventLoop` 上的执行逻辑和 `schedule()` 一样：把任务包装到 Runnable 中，并且在执行时检查是否已经取消，将 Runnable 提交到事件循环上，并且实现取消逻辑。

最后，让我们实验一下：

~~~ java
Scheduler s = new GuiScheduler(new EDTEventLoop());
 
Observable<Integer> source = Observable.just(1)
        .delay(500, TimeUnit.MILLISECONDS, s)
        .doOnNext(v -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            System.out.println(Thread.currentThread());
        });
 
TestSubscriber<Integer> ts1 = new TestSubscriber<>();
TestSubscriber<Integer> ts2 = new TestSubscriber<>();
TestSubscriber<Integer> ts3 = new TestSubscriber<>();
 
source.subscribe(ts1);
source.subscribe(ts2);
source.subscribe(ts3);
 
ts1.awaitTerminalEvent();
ts1.assertNoErrors();
ts1.assertValue(1);
 
ts2.awaitTerminalEvent();
ts2.assertNoErrors();
ts2.assertValue(1);
 
ts3.awaitTerminalEvent();
ts3.assertNoErrors();
ts3.assertValue(1);
~~~

输出应该是这样的：

~~~ java
Thread[AWT-EventQueue-0,6,main]
Thread[AWT-EventQueue-0,6,main]
Thread[AWT-EventQueue-0,6,main]
~~~

## 总结

在关于 `Scheduler` 的最后一篇文章（本文）中，我讲解了如何把事件循环系统的 API 包装为 RxJava 的 Scheduler API。

通常来说，处理 `Scheduler` API 或者我们想要包装的 API 时会遇到一些微妙的问题。这些“单独”的问题很难在本文中进行抽象化，所以如果你有什么有趣或者困难的 API 需要包装，你可以在 [StackOverflow 的 rx-java 话题](http://stackoverflow.com/questions/tagged/rx-java){:target="_blank"}中提问，或者我们的 [Google 群组](https://groups.google.com/forum/#!forum/rxjava){:target="_blank"}中提问，或者直接在本文下评论，以及在 [twitter 上 @akarnokd](https://twitter.com/akarnokd){:target="_blank"} 联系我。

[Reactive-streams](https://github.com/reactive-streams/reactive-streams-io){:target="_blank"} 最近似乎变得越来越知名，但由于它并没有提供太多超出已有的互操作 API 的内容（例如没有 `flatMap`），很多人开始编写一次性的 `Publisher`，并且对它的 `Subscription` 模型的行为产生了疑问。鉴于 RxJava 2.0 将原生支持 reactive-streams API，我们对 `Producer` 的知识将在处理 reactive-streams 的 `Subscription` 时变得很有用。在接下来的系列博客中，我将讲解 reactive-streams API，并且展示如何把 RxJava 的 `Producer` 转换为 `Subscription`。
