---
layout: post
title: 实现操作符时的一些陷阱（三）
tags:
    - Operator
    - Pitfall
---

原文 [Pitfalls of operator implementations (part 3)](http://akarnokd.blogspot.com/2015/05/pitfalls-of-operator-implementations_27.html){:target="_blank"}

## 介绍

在本系列中，我们聚焦于一些常见的，或者比较常见而且很微妙的一些实现操作符的陷阱。既然我们已经知道了更多关于 Producer，subscription 容器类，以及 scheduler 的知识，现在我们可以看更多的陷阱了。

## 9，订阅两次

有些操作符，尤其是那些基于 `OnSubscribe` 的操作符，可能会把它们收到的 `Subscriber` 订阅到其他的 `Observable` 上。`defer()` 就是这样一个例子。

假设你要实现一个操作符，它在把收到的 `Subscriber` 订阅到另一个 `Observable` 上之前，会调用一个回调：

~~~ java
public final class OnSubscribeRunAction<T> 
implements OnSubscribe<T> {
    final Observable actual;
    final Action0 action;
    public OnSubscribeRunAction(Observable actual, Action0 action) {
        this.action = action;
        this.actual = actual;
    }
    @Override
    public void call(Subscriber child) {
        try {
            action.call();
        } catch (Throwable e) {
            child.onError(e);
            return;
        }
        actual.unsafeSubscribe(child);
    }
}
 
Observable<Integer> source = Observable.create(
    new OnSubscribeRunAction<>(Observable.range(1, 3),
() -> {
    System.out.println("Subscribing!");
}));
 
TestSubscriber<Integer> ts = new TestSubscriber<Integer>() {
    @Override
    public void onStart() {
        Thread t = new Thread(() -> {
            System.out.println("Starting helper thread "
                + Thread.currentThread());
        });
        t.start();
    }
};
source.unsafeSubscribe(ts);
~~~

如果运行上面的例子，我们会发现 `onStart` 被调用了两次！问题就出在 RxJava 的 backpressure 相关内容的设计逻辑上：child 被订阅时，它的 `onStart()` 就会被调用，让它有机会在 `onNext` 之前执行它的逻辑。通常来说，这时会发出初始的请求量，或者在 subscriber 开始时显示一些 GUI 的内容。

在普通的“终点 subscriber”（end-subscribers）中，这个问题很少表现出来，因为 `subscribe()` 会把它们包装为一个 `SafeSubscriber`，而后者不会转发 `onStart`。但是当和其他操作符打交道时，`unsafeSubscribe` 是很常见的，因此 `onStart` 就会被执行多次。

解决办法就是在操作符中把 child 包装在一个不会转发 `onStart` 的 subscriber 中：

~~~ java
// ...
try {
    action.call();
} catch (Throwable e) {
    child.onError(e);
    return;
}
actual.unsafeSubscribe(new Subscriber<T>(child) {
    @Override
    public void onNext(T t) {
        child.onNext(t);
    }
    @Override
    public void onError(Throwable e) {
        child.onError(e);
    }
    @Override
    public void onCompleted() {
        child.onCompleted();
    }
});
// ...
~~~

## 10，scheduler worker 泄漏

假设我们在被订阅之后，不是立即执行相关的代码，而是延迟一段时间执行。它的一个使用场景是，如果任务没有按时结束，我们可以做一些优化体验的处理（例如显示一个正在处理的对话框）。

~~~ java
public final class OnSubscribeRunActionDelayed<T>
implements OnSubscribe<T> {
    final Observable actual;
    final Action0 action;
    final long delay;
    final TimeUnit unit;
    final Scheduler scheduler;
    public OnSubscribeRunActionDelayed(Observable actual, 
            Action0 action, long delay, 
            TimeUnit unit, Scheduler scheduler) {
        this.action = action;
        this.actual = actual;
        this.delay = delay;
        this.unit = unit;
        this.scheduler = scheduler;
    }
    @Override
    public void call(Subscriber<? super T> child) {
        SerializedSubscriber<T> s = 
            new SerializedSubscriber<>(child);
         
        Worker w = scheduler.createWorker();                 // (1)
         
        Subscription cancel = w.schedule(() -> {
            try {
                action.call();
            } catch (Throwable e) {
                s.onError(e);
            }
        }, delay, unit);
         
        actual
        .doOnCompleted(cancel::unsubscribe)
        .unsafeSubscribe(s);
    }
}
 
Observable<Integer> source = Observable.create(
    new OnSubscribeRunActionDelayed<>(Observable
        .just(1).delay(1, TimeUnit.SECONDS),
() -> {
    System.out.println("Sorry, it takes too long...");
}, 500, TimeUnit.MILLISECONDS, Schedulers.io()));
 
Subscription s = source.subscribe(System.out::println);
 
Thread.sleep(250);
 
s.unsubscribe();
 
Thread.sleep(1000);
 
source.subscribe(System.out::println);
 
Thread.sleep(1500);
 
for (Thread t : Thread.getAllStackTraces().keySet()) {
    if (t.getName().startsWith("RxCached")) {
            System.out.println(t);
        }
    }
}
~~~

同样，上面代码的执行结果也不符合我们的预期：即便我们取消订阅了第一次 subscription，“Sorry” 还是被打印了出来，而如果我们在最后查看所有的线程，我们会看到两个 `RxCachedThreadScheduler`，但是由于我们进行了复用，显然只应该有一个。

问题就出在 worker 和 schedule 的返回值没有正确参与到取消订阅中：即便我们需要实际订阅的 Observable 很快执行，但取消订阅时，只有 Observable 被取消订阅了，worker 并没有，所以它的线程永远也不会被重新放入线程池中。

这个问题比较微妙，因为 `Schedulers.computation()` 和 `Schedulers.trampoline()` 对 scheduler 的泄漏并不敏感：前者是在一个固定大小的 worker 池中进行调度，后者不会在线程中保持任何资源，所以资源能被正确垃圾回收。但 `Schedulers.io()`，`Schedulers.from()` 和 `newThread()` 正好相反，在 worker 被取消订阅之前，线程都不会被复用或者关闭。

解决办法就是把 worker 和调度结果作为资源添加到 child 中，使得 child 被取消订阅时，能取消订阅所有的资源，但是，由于我们只会在这个 worker 上调度一个任务，所以取消订阅 worker 就会取消“所有”正在执行或者等待执行的任务，所以我们就没必要把调度结果也添加到 child 中，只需要添加 worker 就足够了。

~~~ java
// ...
SerializedSubscriber<T> s = new SerializedSubscriber<>(child);
     
Worker w = scheduler.createWorker(); 
child.add(w);
     
w.schedule(() -> {
    try {
        action.call();
    } catch (Throwable e) {
        s.onError(e);
    }
}, delay, unit);
     
actual
.doOnCompleted(w::unsubscribe)
.unsafeSubscribe(s);
// ...
~~~

## 把 worker 添加到了 subscriber 中

假设我们需要一个这样的操作符，它在收到一个数据时，会发出一个 Observable，这个 Observable 会延迟一定时间发出这个数据之后立即结束。

~~~ java
public final class ValueDelayer<T> 
implements Operator<Observable<T>, T> {
    final Scheduler scheduler;
    final long delay;
    final TimeUnit unit;
     
    public ValueDelayer(long delay, 
            TimeUnit unit, Scheduler scheduler) {
        this.delay = delay;
        this.unit = unit;
        this.scheduler = scheduler;
    }
 
    @Override
    public Subscriber<? super T> call(
            Subscriber<? super Observable<T>> child) {
        Worker w = scheduler.createWorker();
        child.add(w);
         
        Subscriber<T> parent = new Subscriber<T>(child, false) {
            @Override
            public void onNext(T t) {
                BufferUntilSubscriber<T> bus = 
                        BufferUntilSubscriber.create();
                 
                w.schedule(() -> {
                    bus.onNext(t);
                    bus.onCompleted();
                }, delay, unit);
                 
                child.onNext(bus);
            }
            @Override
            public void onError(Throwable e) {
                child.onError(e);
            }
            @Override
            public void onCompleted() {
                child.onCompleted();
            }
        };
         
        child.add(parent);
         
        return parent;
    }
}
 
Observable.range(1, 3)
    .lift(new ValueDelayer<>(1, TimeUnit.SECONDS, 
         Schedulers.computation()))
    .take(1)
    .doOnNext(v -> v.subscribe(System.out::println))
    .subscribe();
         
Thread.sleep(1500);
~~~

奇怪的是，上面的例子不会打印任何内容，但我们希望它会在 1s 之后打印 1。问题就出在 `take(1)` 会在接收到第一个数据之后取消上游 Observable，而这会取消我们延迟执行的任务。

解决这一问题可以有很多种方式，实际上它很依赖于我们的应用场景。显然，这里我们应该取消订阅 worker，但要让内部创建的 Observable 可以被订阅。

一种方式是引入一个原子性的计数器，记录尚未被订阅的内部 Observable 数量，并在计数减到 0 的时候取消订阅 worker。而且，这一方式要求内部的 Observable 一定要被消费。

~~~ java
    // ...
    Worker w = scheduler.createWorker();
     
    final AtomicBoolean once = new AtomicBoolean();
    final AtomicInteger wip = new AtomicInteger(1);           // (1)
     
    Subscriber<T> parent = new Subscriber<T>(child, false) {
        @Override
        public void onNext(T t) {
             
            if (wip.getAndIncrement() == 0) {                 // (2)
                wip.set(0);
                return;
            }
 
            BufferUntilSubscriber<T> bus = 
                    BufferUntilSubscriber.create();
 
            w.schedule(() -> {
                try {
                    bus.onNext(t);
                    bus.onCompleted();
                } finally {
                    release();                                // (3)
                }
            }, delay, unit);
             
            child.onNext(bus);
            if (child.isUnsubscribed()) {
                if (once.compareAndSet(false, true)) {        // (4)
                    release();
                }
            }
        }
        @Override
        public void onError(Throwable e) {
            child.onError(e);
        }
        @Override
        public void onCompleted() {
            if (once.compareAndSet(false, true)) {
                release();                                    // (5)
            }
            child.onCompleted();
        }
        void release() {
            if (wip.decrementAndGet() == 0) {
                w.unsubscribe();
            }
        }
    };
    parent.add(Subscriptions.create(() -> {                   // (6)
        if (once.compareAndSet(false, true)) {
            if (wip.decrementAndGet() == 0) {
                w.unsubscribe();
            }
        }
    }));
     
    child.add(parent);
     
    return parent;
}
// ...
~~~

这一方案有以下几点值得注意：

1. 我们需要一个原子性的整数和布尔值。前者用来记录未被消费的内部 Observable，也就是当前还处于活跃状态的上游 `T` 序列。由于上游可能在很多地方结束，而且可能会结束多次（例如在 `onNext` 之后就终止了，但上游仍发出了 `onCompleted`，而这又会导致一次取消订阅）。由于上游只应该被计数一次，所以我们需要利用 `once` 来保证相应的递减只会发生一次。
2. 在尝试调度任务之前，我们把“活跃窗口”（open windows）计数加一。然而，由于（6）可能异步地把 `wip` 减到 0，所以从 0 增加到 1 就意味着 worker 取消订阅之后发生了 `onNext`，如果这时我们发出一个 `Observable`，它是没有机会被订阅的，所以我们直接返回，不发出 Observable。
3. 一旦内部的 Observable 发射完毕，我们就可以释放一个“窗口”了（递减计数）。
4. 我们在内部的 Observable 被发射出去之后，立即检查 child 是否已经被取消订阅。如果已经被取消订阅，我们尝试取消订阅上游一次。
5. 如果下游没有被取消订阅，那我们就需要在 `onCompleted` 中尝试取消订阅上游一次。
6. 由于 child 的取消订阅可能在任何时刻发生，可能会发生在两次事件发射之间，所以我们依然要在 child 被取消订阅时尝试取消订阅上游一次。

## 总结

大家可能会觉得这些陷阱是一些特殊情况，但作为操作符的开发者，或者打算向 RxJava 提交 PR，大家还是需要了解这些陷阱。

在本文中，我们探讨了可能错误地导致多次订阅 subscriber 的情况，并且展示了把 worker 添加到或者不添加到 child 中会导致问题的两种情况。
