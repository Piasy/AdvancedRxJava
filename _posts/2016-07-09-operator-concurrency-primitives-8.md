---
layout: post
title: Operator 并发原语： producers（六-完结），ProducerObserverArbiter
tags:
    - Operator
    - Producer
---

原文 [Operator concurrency primitives: producers (part 6 - final)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_18.html){:target="_blank"}

## 介绍

你可能已经猜到 `Producer` 是实现操作符的真正大功臣。只要请求和响应数据不是一比一对应，无论是请求多还是数据多，我们都需要引入中间的 producer 来协调请求和响应数据。

在关于 producer 的最后一篇文章中，我将详细讲解最终的通用的 `Producer`，它不仅支持请求处理以及上游 producer 切换，还能保证在上游 producer 切换时事件的传递是串行执行的。在 `switchOnNext()` 中需要这种 producer，因为如果前一个数据源正在发射数据，那我们就不能切换到新的数据源并发射新的数据（_否则就可能并发执行 `onNext` 了_）。我们必须等到上一个数据源正在执行的发射结束，才能进行切换，这样就不会有并发事件，也不会有过度请求。

## The producer-observer-arbiter

解决方案和之前的方案基本一样，利用最初介绍的两种串行访问方式（**发射者循环**或者**队列漏**）实现一个函数，再把所有相关的函数调用都转调到它。下面是这个 producer 的基本类结构：

~~~ java
public final class ProducerObserverArbiter<T> 
implements Producer, Observer<T> {      // (1)
    final Subscriber child;
     
    boolean emitting;
     
    List<Object> queue;                 // (2)
    Producer currentProducer;
    long requested;
     
    public ProducerObserverArbiter(
            Subscriber<? super T> child) {
        this.child = child;
    }
     
    @Override
    public void onNext(T t) {
        // implement
    }
    @Override
    public void onError(Throwable e) {
        // implement
    }
    @Override
    public void onCompleted() {
        // implement
    }
    public void request(long n) {
        // implement
    }
    public void set(Producer p) {
        // implement
    }
    void emitLoop() {
        // implement
    }
}
~~~

基本结构很清晰：我们实现了 `Producer` 和 `Observer`，捕获 child subscriber，并且利用发射者循环来把任务（例如发射数据、处理请求、生产数据）加入队列（2）以完成串行访问。

这里我们没有直接使用各种名为 `missedX` 的成员，而是把它们包装进 holder 类中，这样我们就只需要一个队列就能保存所有的发射事件、请求、producer 切换了：

~~~ java
private static final class ErrorSentinel {                     // (1)
    final Throwable error;
    public ErrorSentinel(Throwable error) {
        this.error = error;
    }
}
 
private static final Object COMPLETED_SENTINEL = new Object(); // (2)
 
private static final class RequestSentinel {                   // (3)
    final long n;
    public RequestSentinel(long n) {
        this.n = n;
    }
}
 
private static final class ProducerSentinel {                  // (4)
    final Producer p;
    public ProducerSentinel(Producer p) {
        this.p = p;
    }
}
~~~

我们定义了 4 种哨兵，这些私有的类或者实例保证了它们内部的数据不会与发射的数据类型 `T` 发生冲突（例如你可以在不干扰 `T` 类型数据流的情况下监听 `Throwable`，`Long` 或者 `Producer` 的事件流）。如果需要支持 `null`，你还可以引入一个 `NULL_SENTINEL`。

1. 我们把错误保存在 `ErrorSentinel` 实例中。
2. 我们用一个常量对象来表示 `onCompleted` 已经发生。
3. 我们把正的请求量和负的生产量保存在 `RequestSentinel` 实例中。
4. 我们把切换或者清除 producer 保存在 `ProducerSentinel` 实例中。

首先让我们实现 `Observer` 的方法：

~~~ java
// ...
@Override
public void onNext(T t) {
    synchronized (this) {
        if (emitting) {
            List<Object> q = queue;
            if (q == null) {
                q = new ArrayList<>();
                queue = q;
            }
            q.add(t);
            return;
        }
        emitting = true;
    }
    boolean skipFinal = false;
    try {
        child.onNext(t);
 
        long r = requested;
        if (r != Long.MAX_VALUE) {            // (1)
            requested = r - 1;
        }
         
        emitLoop();
        skipFinal = true;
    } finally {
        if (!skipFinal) {
            synchronized (this) {
                emitting = false;
            }
        }
    }
}
// ...
~~~

`onNext()` 的实现很直观，而且也和之前的串行方式类似，只不过我们在非无限模式下递减了当前的总请求量（_因为我们立即就执行了一次 `child.onNext()`_）。

~~~ java
// ...
@Override
public void onError(Throwable e) {
    synchronized (this) {
        if (emitting) {
            List<Object> q = new ArrayList<>();
            q.add(new ErrorSentinel(e));        // (1)
            queue = q;
            return;
        }
        emitting = true;
    }
    child.onError(e);                           // (2)
}
// ...
~~~

在 `onError()` 中，（_如果此时有其他线程正在发射_）我们不是把 `Throwable` 加入到之前的事件队列中，而是清空队列再加入（1），所以 `emitLoop()` 将会跳过老的其他事件，优先处理 `onError`。而如果此时没有其他线程正在发射，我们就不必进行循环，也不必把 `emitting` 重置为 `false` 了（2）。

`onCompleted()` 的实现也很类似：

~~~ java
// ...
@Override
public void onCompleted() {
    synchronized (this) {
        if (emitting) {
            List<Object> q = new ArrayList<>();
            q.add(COMPLETED_SENTINEL);
            queue = q;
            return;
        }
        emitting = true;
    }
    child.onCompleted();
}
// ...
~~~

接下来就是 `request()` 的实现了：

~~~ java
// ...
@Override
public void request(long n) {
    if (n < 0) {
        throw new IllegalArgumentException();
    }
    if (n == 0) {
        return;
    }
    synchronized (this) {
        if (emitting) {
            List<Object> q = queue;
            if (q == null) {
                q = new ArrayList<>();
                queue = q;
            }
            q.add(new RequestSentinel(n));          // (1)
            return;
        }
        emitting = true;
    }
    boolean skipFinal = false;
    try {
        long r = requested;
        long u = r + n;
        if (u < 0) {
            u = Long.MAX_VALUE;
        }
        requested = u;                             // (2)
 
        Producer p = currentProducer;
        if (p != null) {                           // (3)
            p.request(n);
        }
        emitLoop();
        skipFinal = true;
    } finally {
        if (!skipFinal) {
            synchronized (this) {
                emitting = false;
            }
        }
    }    
}
// ...
~~~

检查完请求数量的合法性之后，如果现在有其他线程正在发射，我们就把请求数量加入队列（1）。否则我们立即处理，增加总请求量（保证不溢出）（2），如果当前 producer 不为 `null`，我们就向其请求 `n`（3）。

同样 `set()` 方法也是相同的套路：

~~~ java
public void set(Producer p) {
    synchronized (this) {
        if (emitting) {
            List<Object> q = queue;
            if (q == null) {
                q = new ArrayList<>();
                queue = q;
            }
            q.add(new ProducerSentinel(p));
            return;
        }
        emitting = true;
    }
    boolean skipFinal = false;
    try {
        currentProducer = p;
        long r = requested;
        if (p != null && r != 0) {                  // (1)
            p.request(r);
        }
        emitLoop();
        skipFinal = true;
    } finally {
        if (!skipFinal) {
            synchronized (this) {
                emitting = false;
            }
        }
    }
}
~~~

如果新的 producer 不为 `null`，且当前总请求数量非零，我们就向新的 producer 发起请求（1）。

最后，我们终于到了 `emitLoop()` 方法了：

~~~ java
// ...
void emitLoop() {
    for (;;) {
        List<Object> q;
        synchronized (this) {                                    // (1)
            q = queue;
            if (q == null) {
                emitting = false;
                return;
            }
            queue = null;
        }
        long e = 0;
            
        for (Object o : q) {
            if (o == null) {                                     // (2)
                child.onNext(null);
                e++;
            } else if (o == COMPLETED_SENTINEL) {                // (3)
                child.onCompleted();
                return;
            } else if (o.getClass() == ErrorSentinel.class) {    // (4)
                child.onError(((ErrorSentinel)o).error);
                return;
            } else if (o.getClass() == ProducerSentinel.class) { // (5)
                Producer p = (Producer)o;
                currentProducer = p;
                long r = requested;
                if (p != null && r != 0) {
                    p.request(r);
                }
            } else if (o.getClass() == RequestSentinel.class) {  // (6)
                long n = ((RequestSentinel)o).n;
                long u = requested + n;
                if (u < 0) {
                    u = Long.MAX_VALUE;
                }
                requested = u;
                Producer p = currentProducer;
                if (p != null) {
                    p.request(n);
                }
            } else {                                             // (7)
                child.onNext((T)o);
                e++;
            }
        }
        long r = requested;
        if (r != Long.MAX_VALUE) {                               // (8)
            long v = requested - e;
            if (v < 0) {
                throw new IllegalStateException();
            }
            requested = v;
        }
    }
}
~~~

这是目前为止内容最丰富的发射者循环了：

1. 我们一次性把队列中的数据取出来。
2. 对于队列中的每个元素，如果是 `null`，我们就直接把它发给 child（_下游_），并且计一次发射。
3. 如果元素是 `COMPLETED_SENTINEL`，我们就不需要进行后续操作了，就直接退出循环（_`emitting` 不重置为 `false`_）。
4. 否则，我们就可以检查元素类型，来确定需要进行的操作了。如果是 `ErrorSentinel` 类型，我们就把其中错误取出来，发给 child，退出循环。
5. 如果是 `ProducerSentinel`，我们就把其中的 producer 设置给 `currentProducer`，如果它不为 `null`，我们就向它请求当前的累计请求量。
6. 如果是 `RequestSentinel`，我们就把其中的请求量累加到总请求量中（保证不溢出），如果当前 producer 不为 `null`，我们就向它请求新增的量。
7. 如果不是上述任何哨兵类型，那就说明是 `T` 类型的数据，我们就把它发给 child，并计一次发射。
8. 最后，如果 `requested` 不是 `Long.MAX_VALUE`，我们就把它减去这次外循环发射的量并更新。

可能在我们立马就要替换 producer 时仍让老的 producer 继续生产是没有意义甚至不是我们想要的行为。例如在 `switchOnNext` 中，如果连续两次切换数据源，我们可能就希望可以跳过第一个数据源直接前换到第二个数据源。这时我们可以使用在[前文](/AdvancedRxJava/2016/07/02/operator-concurrency-primitives-7/index.html){:target="_blank"}中介绍的 `missedProducer` 方式，而不是把替换 producer 操作加入队列中，我们也可以决定是否需要清空队列中尚未发出的数据。此外，我们也可以使用 `ProducerArbiter` 的成员，来避免处理请求和生产时的额外内存分配。

现在就只差一个使用样例了，现在我们实现一个根据时间在一个固定的 Observable 集合中进行切换的操作符：

~~~ java
public static final class SwitchTimer<T> 
implements OnSubscribe<T> {
    final List<Observable<? extends T>> sources;
    final long time;
    final TimeUnit unit;
    final Scheduler scheduler;
    public SwitchTimer(
            Iterable<? extends Observable<? extends T>> sources, 
            long time, TimeUnit unit, Scheduler scheduler) {
        this.scheduler = scheduler;
        this.sources = new ArrayList<>();
        this.time = time;
        this.unit = unit;
        sources.forEach(this.sources::add);
    }
    @Override
    public void call(Subscriber<? super T> child) {
        ProducerObserverArbiter<T> poa = 
            new ProducerObserverArbiter<>(child);             // (1)
         
        Scheduler.Worker w = scheduler.createWorker();        // (2)
        child.add(w);
         
        child.setProducer(poa);                                  
         
        SerialSubscription ssub = new SerialSubscription();   // (3)
        child.add(ssub);
         
        int[] index = new int[1];
         
        w.schedulePeriodically(() -> {
            int idx = index[0]++;
            if (idx >= sources.size()) {                      // (4)
                poa.onCompleted();
                return;
            }
            Subscriber<T> s = new Subscriber<T>() {           // (5)
                @Override
                public void onNext(T t) {
                    poa.onNext(t);
                }
                @Override
                public void onError(Throwable e) {
                    poa.onError(e);
                }
                @Override
                public void onCompleted() {
                    if (idx + 1 == sources.size()) {          // (6)
                        poa.onCompleted();
                    }
                }
                @Override
                public void setProducer(Producer producer) {
                    poa.set(producer);
                }
            };
 
            ssub.set(s);                                      // (7)
            sources.get(idx).unsafeSubscribe(s);
             
        }, time, time, unit);
    }
}
 
List<Observable<Long>> timers = Arrays.asList(
    Observable.timer(100, 100, TimeUnit.MILLISECONDS),
    Observable.timer(100, 100, TimeUnit.MILLISECONDS)
        .map(v -> v + 20),
    Observable.timer(100, 100, TimeUnit.MILLISECONDS)
        .map(v -> v + 40)
);
 
Observable<Long> source = Observable.create(
    new SwitchTimer<>(timers, 500, 
    TimeUnit.MILLISECONDS, Schedulers.computation()));
         
source.toBlocking().forEach(System.out::println);
~~~

它的结构如下：

1. 我们创建 `ProducerObserverArbiter` 实例。
2. 我们创建了一个 scheduler-worker 实例，并把它加入到 child 的 subscriber 列表中，以便在 child 被取消订阅时可以取消所有的 schedule。
3. 我们需要为 `Observable` 序列保存 `Subscriber` 引用，并把它和 child 级联起来，以便在 child 被取消订阅时可以一同被取消订阅。
4. 如果最后一个 `Observable` 没有及时结束，我们将在再次重复的时候结束整个序列。
5. 否则我们为下一个 `Observable` 创建一个 `Subscriber`（避免和 child 共享状态）。
6. 在 `onCompleted()` 中，我们检查 `idx` 是否是最后一个，如果是我们就调用 `ProducerObserverArbiter#onCompleted()` 来结束序列。
7. `SerialSubscription.set` 会保证在新的 `Observable` 被订阅时，老的订阅会被取消。

## 总结

在 producer 系列文章中，我介绍了从简单如 **single-value-producer** 到复杂如 **producer-observer-arbiter**的众多 producer。随着每个不同的 producer 的介绍，我们涉及到越来越多复杂的内容，也进行了越来越多的说明，希望这些内容可以帮助操作符编写者编写他们需要的解决方案。

接下来关于并发原语的系列，我将介绍各种 `Subscription` 的容器类型，并且会介绍在标准类型无法满足需求时，如何实现自定义的类型。
