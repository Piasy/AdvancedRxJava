---
layout: post
title: SubscribeOn 和 ObserveOn
tags:
    - Operator
    - Scheduler
---

原文 [SubscribeOn and ObserveOn](http://akarnokd.blogspot.com/2016/03/subscribeon-and-observeon.html){:target="_blank"}

注：这篇文章八月份就翻译完成了，当时是为了加深自己对 `subscribeOn` 和 `observeOn` 的理解，本打算按照原文作者的发表顺序发布译文，但今天在写[拆轮子系列：拆 RxJava](/2016/09/15/Understand-RxJava/){:target="_blank"}，里面涉及到了这块内容，为了便于援引，所以提前发布，还好发布在 scheduler 之后也算比较合理。

## 介绍
在响应式编程生态中，最令人疑惑的一对操作符应该就是 `subscribeOn` 和 `observeOn` 了。究其根本原因，可能是以下几点：

+ 它们听起来很像；
+ 从下游来看，有时它们的行为很类似；
+ 它们在某种程度上存在重复；

而名字的疑惑似乎并非 RxJava 独有，Reactor 项目存在类似的问题：`publishOn` 和 `dispatchOn`。显然，不管它们叫什么，大家都很容易对它们产生疑惑。

当我在 2010 年学习 Rx.NET 时，我并未对此产生任何疑惑，`subscribeOn` 影响 `subscribe()`，而 `observeOn` 影响 `onXXX()`。

_（注意：我搜索了早期 Channel 9 的视频，但并未发现类似于本文这样关于这两个操作符工作原理的演讲。内容最接近的大概是[这次演讲了](https://channel9.msdn.com/Blogs/J.Van.Gogh/Controlling-concurrency-in-Rx){:target="_blank"}。）_

我的“论点”是：通过自己实现这样的操作符并分析它们的内部函数调用，我们也许可以消除对它们的疑惑。

## SubscribeOn
`subscribeOn()` 的目的就是，确保调用 `subscribe()` 函数的副作用代码（_执行额外的代码_）在另外的（_参数指定的_）线程中执行。然而首先几乎没有官方的 RxJava 代码会在自己的线程执行这些副作用代码；其次你也可以在自定义的 `Observable` 中执行副作用代码，无论是通过 `create()` 函数来实现，还是通过 `SyncOnSubscribe` 和 `fromCallable()` API 来实现。

那我们为什么需要把副作用代码移到其他的线程中执行呢？主要的使用场景是在当前线程进行网络请求、数据库访问或者其他任何涉及阻塞的操作。让一个 Tomcat 的工作线程阻塞住并不是什么大问题（当然我们也可以通过响应式方式改进这种情况），但在 Swing 应用中阻塞了事件分发线程（Event Dispatch Thread，EDT），或者在安卓中阻塞了主线程，就会对用户体验造成不利的影响了。

_（注：有趣的是，阻塞住 EDT 通常是 GUI 程序中最方便的一种 backpressure 策略，以此防止用户在程序正在执行某项操作的过程中改变程序的状态。）_

因此，如果源头会在被订阅时立即执行一些操作，我们希望这些操作在其他的线程执行。通常我们可以把对 `subscribe()` 的调用以及后续整个过程的所有操作都提交到一个 `ExecutorService` 上，但这时我们会面临取消订阅和 `Subscriber` 分离（cancellation being separate from the `Subscriber`）的问题。随着越来越多（越来越复杂）的操作需要异步地取消订阅，用这样的方式处理所有的情况将会变得很不方便。

幸运的是，我们可以把这一逻辑抽象成一个操作符：`subscribeOn()`。

为了简洁起见，让我们从最简单的响应式类型（Rx.NET 的 `IObservable`）开始构造这个操作符：

~~~ java
@FunctionalInterface
interface IObservable<T> {
    IDisposable subscribe(IObserver<T> observer);
}
 
@FunctionalInterface
interface IDisposable {
    void dispose();
}
 
interface IObserver<T> {
    void onNext(T t);
    void onError(Throwable e);
    void onCompleted();
}
~~~

我们先不考虑同步取消订阅和 backpressure。

我们先假设我们的源头只是简单地休眠 10 秒：

~~~ java
IObservable<Object> sleeper = o -> {
    try {
        Thread.sleep(1000);
        o.onCompleted();
    } catch (InterruptedException ex) {
        o.onError(ex);
    }
};
~~~

显然，当我们调用 `sleeper.subscribe(new IObserver ... )` 时，程序将会休眠 10 秒，现在让我们编写一个操作符，把休眠操作转移到另一个线程执行：

~~~ java
ExecutorService exec = Executors.newSingleThreadedExecutor();
 
IObservable<Object> subscribeOn = o -> {
    Future<?> f = exec.submit(() -> sleeper.subscribe(o));
    return () -> f.cancel(true);
}
~~~

`subscribeOn` 将会提交一个订阅实际 `IObservable` 的任务到 `exec` 上，并且这个任务会返回一个 `IDisposable` 用于取消这个任务的执行。

当然，你也可以通过静态方法，或者是为 `IObservable` 创建一个 wrapper（因为 Java 不支持扩展方法（extension methods）），来实现同样的效果。

~~~ java
public static <T> IObservable<T> subscribeOn(IObservable<T> source, 
    ExecutorService executor);
 
public Observable<T> subscribeOn(Scheduler scheduler);
~~~

关于 `subscribeOn` 最常见的两个问题就是：如果使用了两次 `subscribeOn`（直接或者通过其他操作符间接使用两次），会发生什么？为什么第二次使用 `subscribeOn` 无法再次修改 subscribe 执行的线程？希望通过上述结构上的简化，问题的答案能变得显而易见。首先我们尝试使用两次 `subscribeOn` 操作符：

~~~ java
ExecutorService exec2 = Executors.newSingleThreadedExecutor();
 
IObservable<Object> subscribeOn2 = o -> {
    Future<?> f2 = exec2.submit(() -> subscribeOn.subscribe(o));
    return () -> f2.cancel(true);
};
~~~

现在我们展开一下 `subscribeOn.subscribe()` 的代码：

~~~ java
IObservable<Object> subscribeOn2 = o -> {
    Future<?> f2 = exec2.submit(() -> {
       Future<?> f = exec.submit(() -> {
          sleeper.subscribe(o);
       });
    });
};
~~~

代码很简单，我们可以从头读到尾。当收到 `o` 时，我们提交一个任务到 `exec2` 上，这个任务执行时将会提交另一个任务到 `exec` 上，而这个任务执行时将会用 `o` 订阅 `sleeper`。由于 `subscribeOn2` 是最后使用的，所以无论它将在什么线程被执行，它都是最先执行的，但它一定会被 `subscribeOn` 重新调度（rescheduled） 到 `exec` 上执行。所以源头被订阅之后执行的代码，将在最先使用的（_代码上最靠近源头的_） `subscribeOn()` 操作符指定的线程上执行，而且后续的 `subscribeOn()` 都无法改变这一结果。这就是为什么基于 Rx 的 API 不能在返回 `Observable` 的时候提前使用 `subscribeOn()` 或者提供指定 scheduler 选项的原因。

不幸的是，上述 `subscribeOn` 的实现并没有很好地处理取消订阅：`sleeper.subscribe()` 的返回值并没有和外部的 `IDisposable` 连接起来，所以取消外部的对象并不能“真正地”取消订阅（_译者注：调用最外层返回的 `IDisposable` 的 `dispose()`，会调用 `f2.cancel`，能取消 `f2` 的执行，但这并不会调用 `subscribeOn` 中返回的 `IDisposable` 的 `dispose()`，就更不会调用到 `sleeper.subscribe` 返回的 `IDisposable` 的 `dispose()` 了_）。当然我们可以利用一个组合的（composite） `IDisposable`，把所有的订阅都添加进去，最后一并取消订阅。不过在 RxJava 1.x 中，我们无需如此麻烦，像这样实现这个操作符即可：

~~~ java
Observable.create(subscriber -> {
    Worker worker = scheduler.createWorker();
    subscriber.add(worker);
    worker.schedule(
        () -> source.unsafeSubscribe(Schedulers.wrap(subscriber))
    )
});
~~~

这就保证了 `unsubscribe()` 调用也能取消 `schedule()` 的执行，以及上游使用的任何资源。我们使用 `unsafeSubscribe()` 以避免将原 subscriber 封装为 `SafeSubscriber`，但我们无论如何都需要一次封装，因为 `subscribe()` 和 `unsafeSubscribe()` 都会调用 `Subscriber` 的 `onStart()`，而它很可能已经被外部的 `Observable` 调用过了。所以我们需要避免多次调用用户的 `Subscriber.onStart()` 方法。

上面的代码也实现了 backpressure 支持，但我们还没有完成。

在 RxJava 支持 backpressure 之前，上面的 `subscribeOn()` 实现会保证所有同步的源都会在同一个线程发射所有的数据：

~~~ java
Observable.create(s -> {
    for (int i = 0; i < 1000; i++) {
        if (s.isUnsubscribed()) return;
        
        s.onNext(i);
    }
 
    if (s.isUnsubscribed()) return;
 
    s.onCompleted();
});
~~~

大家都默认地依赖了这一特性。但是 backpressure 打破了这一特性，因为通常情况下调用 `request()` 函数的线程将会执行上面的这个循环（可以看一下 `range()` 的实现），导致可能的线程跳跃（thread-hopping）。所以为了保持这一特性，对 `request()` 的调用必须和原订阅时处于同一个 `Worker`。

所以实际上 `subscribeOn()` 需要进行更多的操作：

~~~ java
subscriber -> {
    Worker worker = scheduler.createWorker();
    subscriber.add(worker);
     
    worker.schedule(() -> {
       Subscriber<T> s = new Subscriber<T>(subscriber) {
           @Override
           public void onNext(T v) {
               subscriber.onNext(v);
           }
 
           @Override
           public void onError(Throwable e) {
               subscriber.onError(e);
           }
 
           @Override
           public void onCompleted() {
               subscriber.onCompleted();
           }
 
           @Override
           public void setProducer(Producer p) {
               subscriber.setProducer(n -> {
                   worker.schedule(() -> p.request(n));
               });
           }
       };
 
       source.unsafeSubscribe(s);
    });
}
~~~

除了转发 `onXXX()` 之外，我们还重写了 `setProducer` 方法，并且通过调度，保证对原 producer 的调用发生在同一个线程，这样就能保证如果 `request()` 调用会导致新的事件发射，它们都会发生在同一个线程。

这里我们可以进行一个小小的优化，我们可以在 schedule 时获取当前的线程，如果调用 `request()` 的线程和这个线程一致，那我们就可以直接调用 `p.request(n)`，省去调度的开销：

~~~ java
Thread current = Thread.currentThread();
 
// ...
 
subscriber.setProducer(n -> {
    if (Thread.currentThread() == current) {
        p.request(n);
    } else {
        worker.schedule(() -> p.request(n));
    }
});
~~~

## ObserveOn
`observeOn` 的目的是确保所有发出的数据/通知都在指定的线程中被接收。RxJava 默认是同步的，即 `onXXX()` 是在同一个线程中串行调用的：

~~~ java
for (int i = 0; i < 1000; i++) {
    MapSubscriber.onNext(i) {
       FilterSubscriber.onNext(i) {
           TakeSubscriber.onNext(i) {
               MySubscriber.onNext(i);
           }
       }
    }
}
~~~

在很多场景下，我们需要把 `onNext()` 的调用（以及其后的所有链式调用）转移到另一个线程中。例如，可能生成 `map()` 的输入是很快的，但是 map 时的计算非常耗时，有可能会阻塞 GUI 线程。又例如，我们可能有些在后台线程中执行的任务（数据库、网络访问，或者耗时的计算），需要把结果在 GUI 中进行展示，很多 GUI 框架只允许在特定线程中修改 GUI 内容。

从概念上来说，`observeOn` 通过调度一个任务，把源 observable 的 `onXXX()` 调度到指定的调度器（scheduler）上。这样，下游接收（执行）`onXXX()` 时，就是在指定的调度器上，但接收的是同样的值：

~~~ java
ExecutorService exec = Executors.newSingleThreadedExecutor();
 
IObservable<T> observeOn = o -> {
    source.subscribe(new Observer<T>() {
        @Override
        public void onNext(T t) {
            exec.submit(() -> o.onNext(t));
        }
         
        @Override
        public void onError(Throwable e) {
            exec.submit(() -> o.onError(e));
        }
 
        @Override
        public void onCompleted() {
            exec.submit(() -> o.onCompleted());
        }            
    });
};
~~~

这种实现方式要求 executor 是单线程的，否则就需要保证 FIFO 以及不会有来自同一个源的多个任务被同时执行。

取消订阅的处理将更加复杂，因为我们必须保持所有正在执行中的任务，当它们执行结束时移除它们，以及保证每个任务都能被及时取消。

我相信 Rx.NET 实现这样的要求需要一套复杂的机制，但幸运的是，RxJava 可以很方便地利用 `Scheduler.Worker` 实现，并达到所有取消订阅需要的功能：

~~~ java
Observable.create(subscriber -> {
    Worker worker = scheduler.createWorker();
    subscriber.add(worker);
 
    source.unsafeSubscribe(new Subscriber<T>(subscriber) {
        @Override
        public void onNext(T t) {
            worker.schedule(() -> subscriber.onNext(t));
        }
         
        @Override
        public void onError(Throwable e) {
            worker.schedule(() -> subscriber.onError(e));
        }
 
        @Override
        public void onCompleted() {
            worker.schedule(() -> subscriber.onCompleted());
        }            
    });
});
~~~

通过对比 `subscribeOn` 和 `observeOn`，我们可以发现 `subscribeOn` 调度了整个 `source.subscribe(...)` 的调用，而 `observeOn` 则是调度每个单独的 `subscriber.onXXX()` 调用。

所以你可以看到如果多次使用 `observeOn`，内部被调度的任务，将会把 `subscriber.onNext` 的执行调度到另一个调度器中：

~~~ java
worker.schedule(() -> worker2.schedule(() -> subscriber.onNext(t)));
~~~

所以 `observeOn` 会重载调用链中指定的线程，因此最靠近 subscriber 的 `observeOn` 指定的线程，将作为最终 `onXXX()` 执行的线程。从上面展开的等效代码我们可以看出，`worker` 被浪费了，因为多余的调度并没有任何意义。

上述的实现方式有一个问题，如果源 observable 是 `range(0, 1M)`，订阅后它会立即发射出所有的数据，所以我们立即会向底层的线程池中提交大量的任务。这会为下游带来压力，同时也会消耗大量的内存。

引入 backpressure 主要就是解决这类问题的：防止内部 buffer 的膨胀，以及由异步执行导致的无限内存占用。消费者通过 `request()` 函数来告诉生产者，它们只能消费多少个数据，以确保生产者只会生产这么多数据（以及调用 `onNext()`）。当消费者准备好之后，它就再次调用 `request()`。上面的 `observeOn()` 实现，通过 `new Subscriber<T>(subscriber)` 包装，它就能够处理 backpressure 了，它将把下游的 `request()` 调用传递给上游。然而它并不能阻止消费者调用 `request(Long.MAX_VALUE)`，此时我们依然存在同样的膨胀问题。

不幸的是，backpressure 的这一问题 RxJava 发现得太晚了，强制要求解决这一问题需要很大的改动。所以，backpressure 作为可选行为被引入，并把解决这一问题作为像 `observeOn` 这样的操作符的责任，以此来保证有限 buffer 的 `Subscriber` 与无限 buffer 的 `Observer` 之间的相同表现（对使用者透明，transparency）。

我们可以通过一个队列、记录下游 `Subscriber` 的请求、向源发送数量固定的请求以及一个队列漏来解决这一问题：

~~~ java
Observable.create(subscriber -> {
    Worker worker = scheduler.createWorker();
    subscriber.add(worker);
 
    source.unsafeSubscribe(new Subscriber<T>(subscriber) {
        final Queue<T> queue = new SpscAtomicArrayQueue<T>(128);
 
        final AtomicLong requested = new AtomicLong();
 
        final AtomicInteger wip = new AtomicInteger();
 
        Producer p;
 
        volatile boolean done;
        Throwable error;
 
        @Override
        public void onNext(T t) {
            queue.offer(t);
            trySchedule();
        }
         
        @Override
        public void onError(Throwable e) {
            error = e;
            done = true;
            trySchedule();
        }
 
        @Override
        public void onCompleted() {
            done = true;
            trySchedule();
        }            
         
        @Override
        public void setProducer(Producer p) {
            this.p = p;
            subscriber.setProducer(n -> {
                BackpressureUtils.addAndGetRequested(requested, n);
                trySchedule();
            }); 
            p.request(128);
        }
 
        void trySchedule() {
            if (wip.getAndIncrement() == 0) {
                worker.schedule(this::drain);
            }
        }
 
        void drain() {
            int missed = 1;
            for (;;) {
                long r = requested.get();
                long e = 0L;
 
                while (e != r) {
                    boolean d = done;
                    T v = queue.poll();
                    boolean empty = v == null;
                     
                    if (checkTerminated(d, empty)) {
                        return;
                    }
 
                    if (empty) {
                        break;
                    }
 
                    subscriber.onNext(v);
                    e++;
                }
 
                if (e == r && checkTerminated(done, queue.isEmpty())) {
                    break;
                }
 
                if (e != 0) {
                    BackpressureHelper.produced(requested, e);
                    p.request(e);
                }
 
                missed = wip.addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }
 
        boolean checkTerminated(boolean d, boolean empty) {
            if (subscriber.isUnsubscribed()) {
                queue.clear();
                return true;
            }
            if (d) {
                Throwable e = error;
                if (e != null) {
                    subscriber.onError(e);
                    return true;
                } else
                if (empty) {
                    subscriber.onCompleted();
                    return true;
                }
            }
            return false;
       }
    });
});
~~~

现在，我们应该对解决方式很熟悉了。我们把上游发射过来的数据加入到队列中，或者保存异常，然后自增 `wip`，并且准备执行队列漏循环。这一过程是必要的，因为可能下游会发起请求，导致上游瞬间发射大量数据。下游发送请求的时候，我们也需要执行队列漏，因为可能此时队列中已经有数据了。在漏循环中，我们会发射队列中的数据，同时也会请求上游 `Producer` 补充数据，上游的 `Producer` 是通过 `setProducer()` 获得的。

我们可以继承（扩展）上面的这个版本，增加额外的安全保护，错误延迟，通过参数控制每次请求的数量，甚至是补充数据的数量。上述 `trySchedule` 的实现具备一个特性：它无需调度器是单线程的。因为 `getAndIncrement` 保证了只会有一个线程能执行队列漏循环，而且只有当 `wip` 递减到零时，才会让其他的线程有机会执行队列漏循环。

## 总结
在本文中，我尝试通过实现一个简单地、不考虑众多复杂情况的版本，来消除对 `subscribeOn` 和 `observeOn` 这两个操作符的疑惑。

我们看到，RxJava 实现中的复杂度，来自于我们需要处理 backpressure，以及对消费者保持透明，无论是否是直接消费序列的消费者。

现在我们理清了这两者的内部实现，（_译者注：接下来这句话实在难懂，很可能错误百出，我将贴出原文，欢迎英语更好的朋友提供更准确的翻译_）接下来我们可以就它们提供的异步边界，继续讨论有关操作符结合（fusion）的话题了。我将以如何结合使用 `subscribeOn` 和 `observeOn` 这两个操作符为例，来展示宏观和微观上的结合能提供怎样的帮助。

Now that the inner workings and structures have been clarified, let's continue with the discussion about operator fusion where I can now use `subscribeOn` and `observeOn` as an example how macro- and micro-fusion can help around the asynchronous boundaries they provide.

以下是我对这个句子的理解：

let's continue with the discussion about operator fusion (1) around the asynchronous boundaries they provide.

(1) where I can now use `subscribeOn` and `observeOn` as an example how macro- and micro-fusion can help
