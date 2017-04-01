---
layout: post
title: Asynchronous Event Streams vs. Reactive-Streams
tags:
    - Reactive Stream
---

原文 [Asynchronous Event Streams vs. Reactive-Streams](http://akarnokd.blogspot.com/2015/11/asynchronous-event-streams-vs-reactive.html){:target="_blank"}。

## 介绍

最近我在 EclipseCon 上看了一场很有意思的演讲，**Asynchronous Event Streams – when java.util.stream met org.osgi.util.promise!**（[演讲视频](https://www.youtube.com/watch?v=urUax2bzKuI)，[规范文档](https://github.com/osgi/design/tree/master/rfcs/rfc0216)）OSGi 的兄弟们也想更好地解决异步编程的问题，也设计了一套自己的 API。它力求实现有限（无限）数据流的处理，错误处理，以及 backpressure 支持。事实证明，没有比这更简单的问题了，所以让我们看看它到底怎么样。

## 接口

前面的规范文档中并没有 Asynchronous Event Streams (AsyncES) 的代码，所以我从它的文档里面提取了代码。

和 Reactive-Streams (RS) 一样，AsyncES 也有对应的概念来代表 Source 和 Consumer。

AsyncES 有 `PushStream` 和 `PushStreamFactory` 接口，前者有类似于 `map`，`flatMap` 之类的 API，后者则是前者的工厂类。比较奇怪的是，`PushStream` 继承了 `Closeable` 和 `AutoCloseable`。因为在规范里面没有任何内容表明 `PushStream` 是 hot stream，所以我们假设它是 cold stream，那这样关闭它们就没有任何意义。

最核心的接口是 `PushEventSource`：

~~~ java
@FunctionalInterface
public interface PushEventSource<T> {
    Closeable open(
        PushEventConsumer<? super T> consumer) throws Exception;
}
~~~

如果大家还记得 `IObservable`/`IObserver`，那这里的模式就应该比较熟悉了。`PushEventSource` 用来指明将向 consumer 生产 T 类型数据的生产者。

这个接口和 `Publisher` 有两个重要的区别：“连接”的方法将返回一个 `Closeable` 对象，而且有可能抛出异常。

遗憾的是，这种模式和 RxJava 早些日子时有同样的问题：

+ 如果是同步的事件流，返回的 Closeable 很难在 consumer 内被取消，除非把 Closeable 传出去（后面介绍）；
+ 这个方法抛出了一个 checked Exception，这会导致代码写起来很不方便，而且也没必要这么做，因为异常可以直接交给 consumer 处理；

在 RS 里面，是否暴露出取消的 API 由 consumer 决定，而且取消操作的工作机制在异步和同步事件流里是一致的。

~~~ java
@FunctionalInterface
public interface PushEventConsumer<T> {
    long ABORT = -1L;
    long CONTINUE = 0L;
     
    long accept(PushEvent<T> e) throws Exception;
}
~~~

接下来就是 consumer 接口了：`PushEventConsumer`。首先它的各种事件并没有单独的方法，只有一个 accept 方法用来接收 `PushEvent` 事件，而且也可能抛出异常，并有返回值。

只有一个 accept 方法让我们想到了 RxJava 里面的 `Notification` 对象，而且的确 `PushEvent` 是一个这样的接口。

另一个有趣的事情是，accept 也可能会抛出异常。如果是在 IO 相关的操作中，这对 lambda 表达式倒是很方便，可以直接抛出而不需要做一层包装，但 `PushEventConsumer` 是最终的消费者，而不是数据源或者中间操作符，异常可以抛到哪里去呢（尤其是在异步的场景中）？

最有意思的部分就在它的返回值类型了，规范里表明它是 backpressue 以及取消操作支持的源头：

+ 负数表明取消或者中止；
+ 0 表明调用方可以立即再次传入新的数据；
+ 正数表明调用方需要等待指定的毫秒数后，才能传入新的数据；

在 RS 中，backpressure 是基于双方协作来实现的。只要下游不请求数据，上游就不会生产/发出数据，消费者可以在任意时刻处理它请求的数据，也可以在任意时刻发出更多的请求。

然而在 AsyncES 中，accept 返回了一个下次调用前需要等待的时间。但我们怎么知道一次 accept 调用会花费多久？而且，如果上游是同步的，那返回正数是否意味着需要调用 `Thread.sleep()`？应该不是。由于没有参考实现，我只能猜测数据的产生利用 `Executor` 实现了某种形式的递归调度：上游在 consumer 到达之后立即安排一次 `PushEvent` 的发射。在 `Runnable` 里调用 accept，并且根据返回值，再延迟 schedule 一个新的 `Runnable`。

这种方式以及接口的设计有以下几点不足：

+ 延迟的单位是固定的（毫秒）；
+ 如果实现使用了 `Executor` 方式，这意味着每一个将要发射的数据都需要一个 `PushEvent` 包装对象，以及一个 `Runnable` 对象（这个开销可不小）；
+ 由于 `ABORT` 是同步返回的，取消功能就没法实现了。如果我们要基于这种结构以及 RxJava 的 `lift()` 方式（consumer 调用其他 consumer 的方法）来实现一个库，如果调用下游的 accept 方法必须切换线程，那我们就无法把返回值传递到原来的线程里面了；

~~~ java
ExecutorService exec = ...
PushEventOperator<T, T> goAsync = downstreamConsumer -> {
    return e -> {
        exec.execute(() -> {
           long backpressure = downstreamConsumer.accept(e);
        });
 
        return backpressure; // ???
    };
}; 
~~~

即便我们用一个 `AtomicLong`，由于代码的执行在另一个线程，除非上游也发出一个数据，否则我们无法返回下游的取消请求，或者下游的 backpressure 延迟请求。

+ 最后一个问题是，如果 consumer 返回的等待时间比它实际需要的等待时间要短（它实际的处理速度没自己认为的那么快），该怎么办？那它只有以下几种选择：默默地接受新的数据，并返回一个更大的延迟；中止处理，并抛出一个异常；或者做无尽缓冲；

我觉得这些问题至少是需要考虑的。RxJava 和 Reactive-Streams 能成现在这个样子是有原因的：找到一种能工作，而且可以组合的异步数据传递方式，并不是一件简单的事情！

## RxJava 和 AsyncES 之间的桥接

让我们编写一个桥接器，给它一个 `Observable`（或者一个 RS 的 `Publisher`），我们把它变成一个 `PushEventSource`：

~~~ java
public static <T> PushEventSource<T> from(
            Publisher<? extends T> publisher, Scheduler scheduler) {
    return c -> {
        // implement
    };
}
~~~

我们利用 T 类型的 Publisher 返回一个 PushEventSource。由于 AsyncES 需要处理等待，所以我们需要一个 Scheduler，以免过快发射数据。由于 PushEventSource 是一个函数式接口，所以我们可以返回一个 lambda 表达式，c 的类型是 `PushEventConsumer<T>`。

接下来让我们一步一步地看实现部分：

~~~ java
CompositeDisposable cd = new CompositeDisposable();
Scheduler.Worker w = scheduler.createWorker();
cd.add(w);
~~~

我们创建一个 CompositeDisposable 和 Worker，并把它们绑定在一起，这样就能实现取消的支持了。

~~~ java
publisher.subscribe(new Subscriber<T>() {
    Subscription s;
    @Override
    public void onSubscribe(Subscription s) {
         // implement
    }
 
    @Override
    public void onNext(T t) {
         // implement
    }
 
    @Override
    public void onError(Throwable t) {
         // implement
    }
 
    @Override
    public void onComplete() {
         // implement
    }
     
});
~~~

接下来我们用一个 Subscriber 订阅到 Publisher 上，在其中我们会把收到的事件进行转换，但在此之前，我们在这里通过 lambda 表达式实现的 open 方法需要返回一个 Closeable：

~~~ java
return cd::dispose;
~~~

到目前为止，整个结构还是比较简单的：lambda 表达式和函数式接口简直行云流水。接下来让我们看看 RS 里面的第一个事件：

~~~ java
@Override
public void onSubscribe(Subscription s) {
    this.s = s;
    cd.add(s::cancel);
    if (!cd.isDisposed()) {
        s.request(1);
    }
}
~~~

首先我们把 Subscription 保存起来，然后把它加入到 CompositeDisposable 中，这样就能一次性取消了，最后如果 cd 没有被取消，我们就向上游请求刚好一个数据。为什么只请求一个数据？因为我们并不清楚 PushEventConsumer 到底想要多少个数据。所以最保险的办法就是只提前请求一个数据，然后在调用 accept 之后，按需延迟之后再请求新的数据：

~~~ java
@Override
public void onNext(T t) {
    long backpressure;
    try {
        backpressure = c.accept(PushEvent.data(t));    // (1)
    } catch (Exception e) {
        onError(e);                                    // (2)
        return;
    }
     
    if (backpressure <= PushEventConsumer.ABORT) {     // (3)
        cd.dispose();
    } else
    if (backpressure == PushEventConsumer.CONTINUE) {  // (4)
        s.request(1);
    } else {
        w.schedule(() -> s.request(1),                 // (5)
             backpressure, TimeUnit.MILLISECONDS);
    }
}
~~~

1. 我们把数据包装为前面提到的 PushEvent 对象（不展开，想想 `rx.Notification`），然后调用 accept；
2. 如果 accept 抛出了异常，我们就调用 onError；
3. 如果 accept 返回了 ABORT，那我们就取消掉 CompositeDisposable（包括 Subscription 和 Worker）；
4. 如果 accept 返回了 CONTINUE (0)，那我们就可以同步请求一个新的数据；
5. 否则我们就需要在指定的毫秒数之后请求一个新的数据；

onError 和 onComplete 事件的处理就比较简单了：取消掉 CompositeDisposable，然后发出一个错误/终止事件的 PushEvent 包装对象：

~~~ java
@Override
public void onError(Throwable t) {
    cd.dispose();
    try {
        c.accept(PushEvent.error(t));
    } catch (Exception ex) {
        RxJavaPlugins.onError(ex);
    }
}
 
@Override
public void onComplete() {
    cd.dispose();
    try {
        c.accept(PushEvent.close());
    } catch (Exception ex) {
        RxJavaPlugins.onError(ex);
    }
 
}
~~~

上面就是 Publisher -> PushEventSource 的转换代码了，实际上，上面的代码并没有什么错误（只是有一点点低效，因为需要各种包装，以及每次只能请求一个数据）。

剩下的就是反向的转换 PushEventSource -> Publisher 了。首先我们看看接口：

~~~ java
public static <T> Observable<T> to(
        PushEventSource<? extends T> source, long backpressure) {
    return Observable.create(s -> {
        // implement
    });
}
~~~

实际上这里 Scheduler 的体现是一个 backpressure 要求等待的毫秒数。我在前面已经提到，我们无法很好地在上下游之间沟通需要的等待时间，或者更具体一点，Subscriber 不知道在重新调用 request() 之前需要等待多久，所以这里接受一个固定的 backpressure 值。

现在让我们逐步看实现代码：

~~~ java
CompositeDisposable cd = new CompositeDisposable();
AtomicLong requested = new AtomicLong();
 
s.onSubscribe(new Subscription() {
    @Override
    public void request(long n) {
        BackpressureHelper.add(requested, n);
    }
     
    @Override
    public void cancel() {
        cd.dispose();
    }
});
~~~

首先我们依然是创建一个 CompositeDisposable，以及用一个 AtomicLong 来记录下游请求的数量。接下来我们用一个 Subscription 来连接下游的请求（通过 BackpressureHelper 累计请求）和取消（取消 CompositeDisposable）方法。

接下来我们建立一个到上游的连接：

~~~ java
try {
    Closeable c = source.open(new PushEventConsumer<T>() {
        @Override
        public long accept(PushEvent<T> c) throws Exception {
            // implement
        }
    });
    cd.add(() -> {
        try {
            c.close();
        } catch (IOException ex1) {
            RxJavaPlugins.onError(ex1);
        }
    });
} catch (Exception ex2) {
    s.onError(ex2);
}
~~~

我们创建了一个 consumer（后文降解），然后向 CompositeDisposable 中加入了一个 Disposable，用来在被取消订阅时调用 open 返回的 Closeable 对象的 close 方法。如果 open 抛出了异常，我们直接把异常发给 Subscriber。

最后我们实现 accept 方法，用来派发 PushEvent：

~~~ java
if (cd.isDisposed()) {                           // (1)
    return ABORT;
}
switch (c.getType()) {                           // (2)
case DATA:
    s.onNext(c.getData());
    for (;;) {
        long r = requested.get();                // (3)
        if (r == 0) {
            return backpressure;                 // (4)
        }
        if (requested.compareAndSet(r, r - 1)) {
            return r > 0L ? 0 : backpressure;    // (5)
        }
    }
case ERROR:                                      // (6)
    cd.dispose();
    s.onError(c.getFailure());
    return ABORT;
case CLOSE:
    cd.dispose();
    s.onComplete();
    return ABORT;
}
return 0L; 
~~~

1. 如果 CompositeDisposable 已经被取消订阅（例如下游取消订阅），那我们就返回 ABORT，它会导致事件流的中止。注意这必须要上游发来一个事件才能触发，我们不能提前触发；
2. 我们根据 PushEvent 的类型来执行不同的处理，如果是 DATA 事件，那就发往下游；
3. 数据发往下游后，我们需要在 CAS 循环中递减请求计数；
4. 如果请求计数为 0，那我们就直接返回最初传入的 backpressure 值；
5. CAS 成功后，如果递减后为 0，那我们也返回 backpressure 值，否则返回 0，告知上游可以立即发出下一个事件；
6. 对于错误和结束事件，我们调用相应的 onXXX 方法即可，然后取消掉 CompositeDisposable；

在上面的实现中有一个无法避免的问题：如果传入的 backpressure 参数不够大怎么办？为了简洁起见，上面的代码并未处理这种情况，所以需要在创建的 Observable 上应用 `onBackpressureXXX` 操作符。实现一个[在前文中提过的支持 `BackpressureStrategy` 的版本](/AdvancedRxJava/2016/10/05/subjects-part-3/)就交给读者了。

既然有了相互转换的方法，那我们就可以轻易地基于 RxJava 2.x（或者其他任何 Reactive-Stream 实现库）实现 PushStream 的方法了。

大家可以在[这个 gist 中](https://gist.github.com/akarnokd/3427a078255cca898c84)看到一个示例，它实现了我们最喜欢的 flatMap 函数。

## 总结

设计一套异步的 API 并不简单，有很多人都在尝试做这件事，而最新的进展就是 OSGi 设计的 Asynchronous Event Streams RFC。

通过本文我们看到，通过等待指定的时间来实现 backpressure，技术上来说可以通过 Reactive-Stream API（尤其是 RxJava）来模拟，所以如果大家想要这样一套机制，可以轻松利用 Reactive-Stream 实现库做到。而反方向的转换就需要更精细地考虑过载问题了。

经过上面的讨论，我比较质疑现在的 Asynchronous Event Stream 能否工作。
