---
layout: post
title: 调度器 Scheduler（二）：自定义 ExecutorService 使用逻辑的 Worker
tags:
    - Scheduler
---

原文 [Schedulers (part 2)](http://akarnokd.blogspot.com/2015/06/schedulers-part-2.html){:target="_blank"}

## 介绍

在[上文中](/AdvancedRxJava/2016/08/05/schedulers-1/index.html){:target="_blank"}，我介绍了如何利用 RxJava 已有的类来实现自定义的 scheduler。

在本文中，我将更加深入一层，演示如何操控底层的 `ExecutorService` 以及 RxJava 其他的基础设施，并与之进行交互，而这些都是无法通过 `NewThreadWorker` 实现的。

## `ScheduledAction` 类

在 `Scheduler`/`Worker` 诞生之前的日子里，和 `Future` 类进行交互是非常直观的：我们只需要用一个 `Subscription` 类包装 `cancel()` 就可以了，这样它就能被加入到各种 subscription 容器类中了。

然而引入了 `Scheduler`/`Worker` 之后就不能这样了，因为我们需要记录这些任务，并且取消它们。一旦 `Future` 被记录了，那就需要在它们完成或者被取消时取消记录，否则就会发生内存泄漏。这一要求就意味着我们不能直接把一个 `Action0`/`Runnable` 提交到 `ExecutorService` 上，我们需要包装一下它们，让它们可以在完成或者被取消时被取消记录。

解决方案就是 `ScheduledAction` 类。所有常规的 `Action0` 都会在 `NewThreadWorker.scheduleActual()` 中被包装为 `ScheduledAction`。它的内部包含了一个 `SubscriptionList`，用来容纳那些在这个任务完成或者取消时要执行的操作：

~~~ java
public final class ScheduledAction
implements Runnable, Subscription {
    final Action0 action;                       // (1)
    final SubscriptionList slist;               // (2)
 
    public ScheduledAction(Action0 action) {
        this.action = action;
        this.slist = new SubscriptionList();
    }
    @Override
    public void run() {
        try {
            action.call();                      // (3)
        } finally {
            unsubscribe();                      // (4)
        }
    }
    @Override
    public boolean isUnsubscribed() {
        return slist.isUnsubscribed();
    }
    @Override
    public void unsubscribe() {
        slist.unsubscribe();
    }
     
    public void add(Subscription s) {           // (5)
        slist.add(s);
    }
}
~~~

这个类还是非常直观的：

1. 我们保存真正要被执行的任务。
2. 我们需要一个容器类来保存所有取消订阅时要执行的任务，由于它只会被增加，所以 `SubscriptionList` 就足够了。
3. 由于 `ExecutorService` 接收的是 `Runnable`，所以我们实现 `Runnable` 接口，并在 `run()` 函数中执行实际的任务。
4. 无论任务执行成功与否，我们都调用 `unsubscribe()` 函数，用来触发执行清理任务。
5. 但是这些清理任务需要注册到 `ScheduledAction` 中，所以我们暴露出 `SubscriptionList` 的 `add()` 函数。

接下来就是要在任务被提交到 `ExecutorService` 之前把所有的记录以及清理任务都串起来。为了简单起见，我们假设我们的 ExecutorService 是个单线程的 service。我们将在后面处理多线程的情况。首先让我们看一下自定义 `Worker` 的结构：

~~~ java
public final class CustomWorker 
extends Scheduler.Worker {
    final ExecutorService exec;                             // (1)
    final CompositeSubscription tracking;                   // (2)
    final boolean shutdown;                                 // (3)
     
    public CustomWorker() {
        exec = Executors.newSingleThreadExecutor();
        tracking = new CompositeSubscription();
        shutdown = true;
    }
    public CustomWorker(ExecutorService exec) {
        this.exec = exec;
        tracking = new CompositeSubscription();
        shutdown = false;                                   // (4)
    }
    @Override
    public Subscription schedule(Action0 action) {
        return schedule(action, 0, null);                   // (5)
    }
    @Override
    public Subscription schedule(Action0 action,
            long delayTime, TimeUnit unit) {
        // implement
    }
    @Override
    public boolean isUnsubscribed() {
        return tracking.isUnsubscribed();                   // (6)
    }
    @Override
    public void unsubscribe() {
        if (shutdown) {
            exec.shutdownNow();                             // (7)
        }
        tracking.unsubscribe();
    }
}
~~~

目前为止，这个结构还不复杂：

1. 我们保存实际的线程池的引用。
2. 我们还需要保存提交过来的任务的引用，以便于后面取消它们。
3. 我们也打算支持用户使用自己的单线程 service，在这种情况下，关闭 service 就是调用者的责任了。
4. 我们只关闭我们自己创建的 service，不关闭在构造函数中传入的 service。
5. 我们把无延迟的调度转发为一个延迟为 0 的调度。
6. `tracking` 成员还记录被提交的任务是否已经被取消。
7. 如果 `ExecutorService` 是我们自己创建的，那我们就将它关闭，然后取消订阅所有提交的任务（注意，如果 service 是我们自己创建的，那其实我们不需要记录提交过来的任务，因为 service 已经记录了，我们可以在 `shutdownNow()` 中取消订阅它们）。

最后，我们看一下延迟 `schedule()` 的实现：

~~~ java
// ...
@Override
public Subscription schedule(Action0 action, 
        long delayTime, TimeUnit unit) {
    if (isUnsubscribed()) {                                // (1)
        return Subscriptions.unsubscribed();
    }
    ScheduledAction sa = new ScheduledAction(action);      // (2)
     
    tracking.add(sa);                                      // (3)
    sa.add(Subscriptions.create(
        () -> tracking.remove(sa)));
     
    Future<?> f;
    if (delayTime <= 0) {                                  // (4)
        f = exec.submit(sa);
    } else if (exec instanceof ScheduledExecutorService) { // (5)
        f = ((ScheduledExecutorService)exec)
             .schedule(sa, delayTime, unit);
    } else {
        f = genericScheduler.schedule(() -> {              // (6)
            Future<?> g = exec.submit(sa);
            sa.add(Subscriptions.create(                   // (7)
                () -> g.cancel(false)));
        }, delayTime, unit);
    }
     
    sa.add(Subscriptions.create(                           // (8)
        () -> f.cancel(false)));
 
    return sa;                                             // (9)
}
// ...
~~~

本文第一段复杂的代码工作机制如下：

1. 如果 worker 已经被取消订阅，我们就返回一个表示已经取消的常量 subscription。注意，如果有 schedule 调用（_由于多线程竞争_）通过了这个检查，它将会收到一个来自底层线程池的 `RejectedExecutionException`。你可以把函数中后面的代码都用一个 `try-catch` 包裹起来，并在异常发生时返回同样的表示已经取消的常量 subscription。
2. 我们把任务包装为 `ScheduledAction`。
3. 在实际调度这个任务之前，我们把它加入到 `tracking` 中，并且增加一个取消订阅的回调，以便在它执行完毕或者被取消时可以将其从 `tracking` 中移除。注意，由于幂等性，`remove()` 不会调用 `ScheduledAction` 的 `unsubscribe()`，从而不会导致死循环。
4. 如果调度是没有延迟的，我们就立即将其提交，并且保存返回的 `Future`。
5. 如果我们的 `ExecutorService` 是 `ScheduledExecutorService`，我们就可以直接调用它的 `schedule()` 函数了。
6. 否则我们就需要借助 `ScheduledExecutorService` 来实现延迟调度了，但我们不能直接把任务调度给它，因为这样它会在错误的线程中执行。我们需要创建一个中间任务，它将在延迟结束之后，向正确的线程池调度一个即时的任务。
7. 我们需要保证提交后返回的 `Future` 能在 `unsubscribe()` 调用时被取消。这里我们把内部的 `Future` 加入到了 `ScheduledAction` 中。
8. 无论是立即调度，还是延迟调度，我们都需要在取消订阅时取消这个调度，所以我们把返回的 `Future` 加入到 `ScheduledAction` 中（_通过把 `Future#cancel()` 包装到一个 `Subscription` 中_）。在这里，你就可以控制是否需要强行（_中断_）取消了。（RxJava 会根据取消订阅时所处的线程来决定：如果取消订阅就是在执行任务的线程中，那就没必要中断了）
9. `ScheduledAction` 也是任务发起方用来取消订阅的凭证（token）。

由于 subscription 容器类的终结状态特性，即便（7）和（8）发生在 `ScheduledAction` 被取消订阅之后，它们也会立即被取消订阅。至于更加激进的需求，你可以在 `ScheduledAction#run()` 中在执行实际任务之前检查是否已经被取消订阅。

最后缺失的一点代码就是 `genericScheduler` 了。你可以为 worker 添加一个 `static final` 成员，并像下面这样设置：

~~~ java
// ...
static final ScheduledExecutorService genericScheduler;
static {
    genericScheduler = Executors.newScheduledThreadPool(1, r -> {
        Thread t = new Thread(r, "GenericScheduler");
        t.setDaemon(true);
        return t;
    });
}
// ...
~~~

## 结论

在本文中，我演示了如何把一个任务包装为一个可以被调度的任务，并且如何让它们可以在单个任务层面以及整个 worker 层面被取消。

在本系列的最后一篇文章中，我将讲解如何处理多线程的 `ExecutorService`，因为我们不能让非延迟调度的任务执行时乱序甚至并发执行。
