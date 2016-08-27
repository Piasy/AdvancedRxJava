---
layout: post
title: 调度器 Scheduler（三）：包装多线程 Executor
tags:
    - Scheduler
---

原文 [Schedulers (part 3)](http://akarnokd.blogspot.com/2015/06/schedulers-part-3.html){:target="_blank"}

## 介绍

在本文中，我将讲讲如何把已有的多线程 `Executor` 包装为一个 scheduler，并且遵循 `Scheduler` 和 `Worker` 的规则。

`Worker` 最重要的一个规则就是有序提交的非延迟任务要按序执行，但是 `Executor` 的线程是随机取走任务，而且是并发乱序执行的。

解决办法就是使用我们以前介绍过的“队列漏”，并且对调度的任务进行一个中继操作（_原文是 trampolining，意为“蹦床”，但没懂是什么意思，推测是“中继”吧_）。队列会保证任务提交的顺序得到了保存，而漏的逻辑则保证了任意时刻最多只会有一个任务执行，不会出现并发执行。

## `ExecutorScheduler`

首先，让我们看看类结构：

~~~ java
public final class ExecutorScheduler extends Scheduler {
    final Executor exec;
    public ExecutorScheduler(Executor exec) {
        this.exec = exec;
    }
    @Override
    public Worker createWorker() {
        return new ExecutorWorker();
    }
     
    final class ExecutorWorker extends Worker 
    implements Runnable {                             // (1)
        // data fields here
        @Override
        public Subscription schedule(
                Action0 action) {
            // implement
        }
        @Override
        public Subscription schedule(
                Action0 action, long delayTime,
                TimeUnit unit) {
            // implement
        }
        @Override
        public void run() {
            // implement
        }
        @Override
        public boolean isUnsubscribed() {
            // implement
        }
        @Override
        public void unsubscribe() {
            // implement
        }
    }
}
~~~

所有的 worker 实例都会把任务转发到同一个底层的 `Executor` 上。需要注意的是，我们让 `ExecutorWorker` 实现了 `Runnable`，这样我们就可以在后面的漏逻辑中省去创建一个单独的 `Runnable` 实例。

漏逻辑需要一个队列、一个正在执行的标记、以及一个 subscription 容器类型，以便集中取消订阅：

~~~ java
// ...
final AtomicInteger wip = new AtomicInteger();
final Queue<ScheduledAction> queue = new ConcurrentLinkedQueue<>();
final CompositeSubscription tracking = new CompositeSubscription();
// ...
~~~

这里使用了 `ConcurrentLinkedQueue`，可能会让一些性能追求者（例如我）不满意，因为有可能我们只需要 [JCTools](https://github.com/JCTools/JCTools){:target="_blank"} 中的 `MpscLinkedQueue`，甚至是我实现的 [MpscLinkedArrayQueue](https://gist.github.com/akarnokd/0699387c4ed43f5d4386){:target="_blank"}。

这里需要权衡：我们是否愿意承担已取消任务仍被保留的问题。在基于 `ScheduledExecutorService` 的 `Scheduler` 中这并不是问题，因为它们会自动将已取消的任务从内部队列中移除（或者在 Java 6 的环境中定期清理）。但是这些删除操作在 JCTools 中都不能用，所以目前来看，如果不允许已取消任务仍被保留，最好的选择就是使用 `ConcurrentLinkedQueue`（当然，我们也可以实现一个特定的队列，以及特定的任务类，使得每个任务都知道其他任务的状态，所以取消订阅时可以定位到这个任务，然后用一个 CAS 操作将其替换为一个“墓碑”任务，_一个特殊的用来标记已取消的任务_）。但是要注意，这里移除操作的复杂度是 O(n)。

基于上面的数据成员，让我们先实现几个简单地函数：

~~~ java
        // ...
        @Override
        public boolean isUnsubscribed() {
            return tracking.isUnsubscribed();
        }

        @Override
        public void unsubscribe() {
            queue.clear();
            tracking.unsubscribe();
        }
    }
}
~~~

注意，我们无权控制 `Executor` 的声明周期，所以不应该结束它，而且一旦我们结束了 `Executor`，其他的 worker 也就会停止工作了。最好的方式就是记录提交到这个 worker 的任务，并且将它们一起取消。

现在开始复杂部分了，首先让我们看看无延迟的 `schedule()`：

~~~ java
@Override
public Subscription schedule(Action0 action) {
    if (isUnsubscribed()) {
        return Subscriptions.unsubscribed();
    }
    ScheduledAction sa = 
            new ScheduledAction(action);
    tracking.add(sa);
    sa.add(Subscriptions.create(
            () -> tracking.remove(sa)));        // (1)
        
    queue.offer(sa);                            // (2)
         
    sa.add(Subscriptions.create(
            () -> queue.remove(sa)));           // (3)
         
    if (wip.getAndIncrement() == 0) {           // (4)
        exec.execute(this);                     // (5)
    }
         
    return sa;
}
~~~

我们现在不能简单地将调用转发到延迟版本中了，它需要自己的逻辑：

1. 我们创建一个 `ScheduledAction`（在[第二部分中实现的](/AdvancedRxJava/2016/08/19/schedulers-2/){:target="_blank"}），并且在其被取消订阅时将自己从 `tracking` 中移除。
2. 我们把任务加入到队列中，队列会保证任务按照提交顺序先进先出（FIFO）执行。
3. 我们在任务被取消时，将任务从队列中移除。注意这里的移除操作复杂度为 O(n)，n 表示队列中在该任务之前等待执行的任务数。
4. 我们只允许一个漏线程，也就是把 `wip` 从 0 增加到 1 的线程。
5. 如果当前线程赢得了漏权利，那我们就把 worker 自己提交到 `Executor` 上，并在 `run()` 函数中从队列中取出任务执行。

注意，即便我们持有了 `ExecutorService` 的引用，我们也不能把这次提交的 `Future` 和任何 action 联系起来，所以取消任务只能通过间接的方式完成。

现在我们继续实现 `run()` 函数：

~~~ java
@Override
public void run() {
    do {
        if (isUnsubscribed()) {                   // (1)
            queue.clear();
            return;
        }
        ScheduledAction sa = queue.poll();        // (2)
        if (sa != null && !sa.isUnsubscribed()) {
            sa.run();                             // (3)
        }
    } while (wip.decrementAndGet() > 0);          // (4)
}
~~~

漏的逻辑比较直观：

1. 我们先检查 worker 是否已经被取消请阅，如果已经取消，那我们就清空队列，并且返回。
2. 我们从队列中取出一个任务。
3. 由于在 `run()` 函数执行期间可能会删除任务，或者由于取消订阅而清空队列，所以我们需要检查是否为 null，以及是否已经被取消订阅。如果都没有，那我们就执行这个任务。
4. 我们递减 `wip`，直到为 0 就退出循环，此时我们就可以安全地重新调度这个 worker，并在 `Executor` 上执行漏任务了。

最后，最复杂的就是延迟调度的实现了 `schedule()`：

~~~ java
@Override
public Subscription schedule(
        Action0 action, 
        long delayTime,
        TimeUnit unit) {
 
    if (delayTime <= 0) {
        return schedule(action);                      // (1)
    }
    if (isUnsubscribed()) {
        return Subscriptions.unsubscribed();          // (2)
    }
     
    ScheduledAction sa = 
            new ScheduledAction(action);
    tracking.add(sa);
    sa.add(Subscriptions.create(
            () -> tracking.remove(sa)));              // (3)
     
    ScheduledExecutorService schedex;
    if (exec instanceof ScheduledExecutorService) {
        schedex = (ScheduledExecutorService) exec;    // (4)
    } else {
        schedex = CustomWorker.genericScheduler;      // (5)
    }
     
    Future<?> f = schedex.schedule(() -> {            // (6)
         
        queue.offer(sa);                              // (7)
         
        sa.add(Subscriptions.create(
                () -> queue.remove(sa)));
         
        if (wip.getAndIncrement() == 0) {
            exec.execute(this);
        }
         
    }, delayTime, unit);
     
    sa.add(Subscriptions.create(
            () -> f.cancel(false)));                  // (8)
     
    return sa;
}
~~~

看上去和非延迟调度比较像，除了要想办法实现延迟：

1. 为了避免额外的开销，如果延迟非正，那我们就直接转发给非延迟的 `schedule()`。
2. 如果 worker 已经被取消订阅，我们直接返回表示已经取消订阅的 `Subscription`。
3. 我们把任务包装为 `ScheduledAction`，加入到 `tracking` 中，并且在取消订阅时从 `tracking` 中移除。
4. 我们需要一个支持延迟执行的 `Executor`，所以我们检查一下我们的 `Executor` 是否支持这一功能（`ScheduledExecutorService`）。
5. 如果不支持，那我们就要自己来实现延迟执行，例如使用[本系列第二篇](/AdvancedRxJava/2016/08/19/schedulers-2/){:target="_blank"}中的 `CustomWorker.genericScheduler`。
6. 有了支持延迟执行的 service 之后，我们直接延迟执行任务即可。
7. 当延迟任务执行时，我们把任务加入到队列中，并在取消订阅时将任务从队列中移除，递增 `wip`，并且在 0 到 1 的递增情况下进入漏循环。
8. 我们需要保证在取消订阅时，也要取消掉我们延迟执行的任务（通过返回的 `Future`）。

让我们实验一下 `ExecutorScheduler`：

~~~ java
ExecutorService exec = Executors.newFixedThreadPool(3);
try {
    Scheduler s = new ExecutorScheduler(exec);
 
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
} finally {
    exec.shutdown();
}
~~~

我们将得到这样的输出：

~~~ bash
Thread[pool-1-thread-3,5,main]
Thread[pool-1-thread-1,5,main]
Thread[pool-1-thread-2,5,main]
~~~

_译者注：这个例子是这样的，`delay` 操作传入了我们实现的 `scheduler`，每次 subscribe 时，都会创建一个新的 worker，我们连续 subscribe 了 3 次，每次执行时都会 sleep 1 秒，所以我们每次 subscribe 都会执行在不同的线程上。但这个例子并没有验证在同一个 worker 上进行多次 schedule 能保证串行执行，所以这个例子并不恰当。_

## 总结

在这篇关于 `Scheduler` 的文章中，我讲解了在底层 `Executor` 是多线程时，如何保证非延迟调度串行按序执行。

在下一篇中，我将讲解如何在不支持像 `Future` 这样的取消机制的框架中（例如 GUI 框架的事件循环），让 `Scheduler` 和它们很好地一起工作。
