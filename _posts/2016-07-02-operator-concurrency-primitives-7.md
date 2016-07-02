---
layout: post
title: Operator 并发原语： producers（五），ProducerArbiter
tags:
    - Operator
    - Producer
---

原文 [Operator concurrency primitives: producers (part 5)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_15.html){:target="_blank"}

## 介绍

在编写操作符的时候，同时处理多个数据源以及 backpressure 并非易事。即便简单如在第一个 `Observable` 发射结束之后继续发射第二个 `Observable` 的数据这一功能，要正确处理好请求处理和传播，也是存在很大挑战的。

通常在 RxJava 中，尤其是 [reactive-streams](https://github.com/reactive-streams/reactive-streams-jvm){:target="_blank"}，我们必须且只能为 `Subscriber` 设置一次 `Producer`。不要以为 `Subscriber.setProducer()` 具备线程安全性，它就能被随时改变，因为 `Subscriber` 不负责（我认为也不应该负责）处理请求，它只负责简单地记录请求数目，并在 `Producer` 被设置时把请求计数转发给 producer。如果你请求了很多数据，并且中途替换了 producer，那新的 producer 将会收到在它被设置时 subscriber 的成员 `requested` 记录的总请求量，而不是老的 producer 仍未处理的请求量。

在本文中，我将介绍一种能够帮助我们应对这一挑战的 producer。

## producer-arbiter

假设我们需要编写这样一个操作符（注意我们并不是要 `concatWith()`）：给定两个 `Observable`，它将观察第一个 `Observable`，当第一个 `Observable` 结束时它将开始观察第二个 `Observable`，当第二个 `Observable` 结束时，它才会结束。

~~~ java
public final class ThenObserve<T> implements Operator<T, T> {
    final Observable<? extends T> other;
    public ThenObserve(Observable<? extends T> other) {
        this.other = other;
    }
 
    @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        Subscriber<T> parent = new Subscriber<T>(child, false) {
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
                other.unsafeSubscribe(child);
            }
        };
        child.add(parent);
        return parent;
    }
}
 
Observable<Integer> source = Observable
    .range(1, 10)
    .lift(new ThenObserve<>(Observable.range(11, 90)));
 
source.subscribe(System.out::println);
 
System.out.println("---");
 
TestSubscriber<Integer> ts = new TestSubscriber<>();
ts.requestMore(20);
 
source.subscribe(ts);
 
ts.getOnNextEvents().forEach(System.out::println);
~~~

如果你运行上面的代码，第一次订阅 `source` 将会打印正确的内容：从 1 到 100。但第二次订阅，尽管我们只请求了 20 个，但它却打印了从 1 到 30！显然，第二个 `range` observable 并不是只发出剩下的 10 个数据，而是发出了 20 个数据。

因此我们需要记录下游请求的数据量，以及上游发出的数据量，一旦数据源发生变化，我们要让新数据源只发出剩下的数据量。为了实现这一功能，我们需要一个能够线程安全地管理请求和数据源切换的 producer。

让我们称之为 `ProducerArbiter`，其基本结构如下：

~~~ java
public final class ProducerArbiter 
implements Producer {
    long requested;                                   // (1)   
    Producer currentProducer;

    boolean emitting;                                 // (2)
    long missedRequested;
    long missedProduced;
    Producer missedProducer;
    
    static final Producer NULL_PRODUCER = n -> { };   // (3)
    
    @Override
    public void request(long n) {
        // to implement
    }
    
    public void produced(long n) {
        // to implement
    }
    
    public void set(Producer newProducer) {
        // to implement
    }

    public void emitLoop() {
        // to implement
    }
}
~~~

`ProducerArbiter` 看起来并无特殊之处，目前看来，它有以下几个特性：

1. 我们记录到目前为止总的请求数量，也记录当前的 producer。当前的 producer 可能为 `null`，这种情况下我们依然累计总请求数量，并在 producer 被设置时，一起发送给 producer。
2. 我们将使用 [发射者循环（emitter-loop）](/AdvancedRxJava/2016/05/06/operator-concurrency-primitives/){:target="_blank"}方式来实现串行访问，因为我们需要支持并发调用 `request()`，以及并发切换 producer。_注意，这并不是保证从上游到下游的 `onNext` 事件串行发生，并发切换 producer 可能会导致来自多个上游 observable 的 `onNext` 事件同时发生。我将在本系列后续的文章中解决这一问题，不过幸运的是，在本文的 `ThenObserve` 中，这个问题并没有影响。_
3. 有时我们需要清除对当前 producer 的引用，但我们用 `missedProducer` 为 `null` 来代表当前没有设置 producer 的尝试。

有了这个基本结构之后，让我们逐个实现上面的函数：

~~~ java
// ... same as before
@Override
public void request(long n) {
    if (n < 0) {                                   // (1)
        throw new IllegalArgumentException();
    }
    if (n == 0) {
        return;
    }
    synchronized (this) {
        if (emitting) {
            missedRequested += n;                  // (2)
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
        requested = u;                             // (3)
        
        Producer p = currentProducer;
        if (p != null) {
            p.request(n);                          // (4)
        }
        
        emitLoop();                                // (5)
        skipFinal = true;
    } finally {
        if (!skipFinal) {                          // (6)
            synchronized (this) {
                emitting = false;
            }
        }
    }
}
~~~

别怕，这个函数不是本文中最长的函数：

1. 我们进行普通的请求数量检查。
2. 我们把错过的请求（_还未来得及处理的请求_）单独记录在 `missedRequested` 中（_而非记录在 `missedProduced` 中_），因为如果把它们记录在一起，我们就无法区分被允许的请求“溢出”（当成是请求 `Long.MAX_VALUE`）和过度生产了。
3. 我们也和其他 producer 一样更新 `requested`，并将其限制在 `Long.MAX_VALUE` 中。
4. 如果当前已经设置了 producer，我们就请求 `n` 个（注意并不是 `requested` 个！），因为请求是在每个 producer 上进行累计的。
5. `request` 可能被并发调用，所以我们进入发射者循环的循环阶段中。
6. 如果 `emitLoop()` 正常返回，我们跳过 `finally` 语句块，否则我们需要清除 `emitting` 标记，使得后续的调用可以继续利用 ProducerArbiter（可能是不同的 producer，可以参见 `Observable.retry()`）。

接下来是 `produced` 函数：

~~~ java
// ... continued
public void produced(long n) {
    if (n <= 0) {                                    // (1)
        throw new IllegalArgumentException();
    }
    synchronized (this) {
        if (emitting) {
            missedProduced += n;                     // (2)
            return;
        }
        emitting = true;
    }
    
    boolean skipFinal = false;
    try {
        long r = requested;
        long u = r - n;
        if (u < 0) {
            throw new IllegalStateException();       // (3)
        }
        requested = u;
    
        emitLoop();                                  // (4)
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

结构和 `request` 比较类似：

1. 如果已生产的数量非正，则说明出了 bug。
2. 我们单独记录错过的已生产数量（请看 `request()` 中的解释）。
3. 我们从 `requested` 中减去 `n`，并检查下溢。下溢意味着 bug 或者上游没有支持 backpressure。
4. 我们尝试处理所有错过的事件。

~~~ java
// ... continued
public void set(Producer newProducer) {
    synchronized (this) {
        if (emitting) {
            missedProducer = newProducer == null ? 
                NULL_PRODUCER : newProducer;          // (1)
            return;
        }
        emitting = true;
    }
    boolean skipFinal = false;
    try {
        currentProducer = newProducer;
        if (newProducer != null) {
            newProducer.request(requested);           // (2)
        }
        
        emitLoop();                                   // (3)
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

代码结构和行为也是很相似的：

1. 当前的 producer 要么被更新，要么被清除。我们直接覆盖了之前的错过的 producer，因为反正它们也没有机会可以发出数据。
2. 如果我们获得了新的 producer，我们直接请求 `requested` 个数据。这保证了新的 producer 将生产之前的 producer 未生产完毕的数量。_注意，在 `onNext` 事件的同时切换 producer 可能造成多余请求：新的 producer 会收到 n 的请求，但老的 producer 可能会在停止之前发出一个数据，这样就多了一个数据，而且有可能导致 `MissingBackpressureException`。正如前文所述，在本文的情况下，这一问题不会发生，后面我将单独成文解决这一问题。_
3. 剩下的代码就和其他部分一样了：进入循环，处理错过的事件，并处理好可能的异常。

最后就是 `emitLoop()` 函数了：

~~~ java
// ... continued
public void emitLoop() {
    for (;;) {
        long localRequested;
        long localProduced;
        Producer localProducer;
        synchronized (this) {
            localRequested = missedRequested;
            localProduced = missedProduced;
            localProducer = missedProducer;
            if (localRequested == 0L 
                    && localProduced == 0L
                    && localProducer == null) {       // (1)
                emitting = false;
                return;
            }
            missedRequested = 0L;
            missedProduced = 0L;
            missedProducer = null;                    // (2)
        }
        
        long r = requested;
        
        if (r != Long.MAX_VALUE) {                    // (3)
            long u = r + localRequested;
            if (u < 0 || u == Long.MAX_VALUE) {       // (4)
                r = Long.MAX_VALUE;
                requested = r;
            } else {
                long v = u - localProduced;           // (5)
                if (v < 0) {
                    throw new IllegalStateException();
                }
                r = v;
                requested = v;
            }
        }
        if (localProducer != null) {                  // (6)
            if (localProducer == NULL_PRODUCER) {
                currentProducer = null;
            } else {
                currentProducer = localProducer;
                localProducer.request(r);             // (7)
            }
        } else {
            Producer p = currentProducer;
            if (p != null && localRequested != 0L) {
                p.request(localRequested);            // (8)
            }
        }
    }
}
~~~

在 `emitLoop()` 中我们需要处理多种状态和可能：

1. 退出循环的条件是没有任何待处理的请求、已生产数量更新以及 producer 切换。
2. 如果我们还需要继续循环，我们首先清除各个标记成员。
3. 如果当前请求的数量是 `Long.MAX_VALUE`，那我们处理请求时就可以简化了，因为下游处于无限请求模式。
4. 即便当前请求的数量不是无限，加上待处理的请求之后也可能是无限，这种情况下也可以简化处理。
5. 如果仍不是无限请求模式，那我们就需要减去已生产的数量，更新计数的局部变量以及成员变量。
6. 如果需要切换 producer，我们就更新 `currentProducer`（或者清除）。
7. 如果新的 producer 没有被清除，我们就更新 `currentProducer`，并一次性请求截至当前所需要请求的数量。
8. 如果不切换 producer，那我们只需要请求新增的待处理请求（如果 producer 不为 `null` 的话）。

实现了完整的 `ProducerArbiter` 之后，我们就可以修改我们的 `ThenObserve` 使之能正常工作了：

~~~ java
public final class ThenObserve<T> implements Operator<T, T> {
    final Observable<? extends T> other;
    public ThenObserve(Observable<? extends T> other) {
        this.other = other;
    }

    @Override
    public Subscriber<? super T> call(
            Subscriber<? super T> child) {
        ProducerArbiter pa = new ProducerArbiter();         // (1)
        
        Subscriber<T> parent = new Subscriber<T>() {
            @Override
            public void onNext(T t) {
                child.onNext(t);
                pa.produced(1);                             // (2)
            }
            @Override
            public void onError(Throwable e) {
                pa.set(null);                               // (3)
                child.onError(e);
            }
            @Override
            public void onCompleted() {
                pa.set(null);                               // (4)
                
                Subscriber<T> parent2 = create2(pa, child); // (5)
                child.add(parent2);
                
                other.unsafeSubscribe(parent2);             // (6)
            }
            @Override
            public void setProducer(Producer producer) {
                pa.set(producer);                           // (7)
            }
        };
        child.add(parent);
        child.setProducer(pa);
        return parent;
    }
    
    Subscriber<T> create2(ProducerArbiter pa, 
            Subscriber<? super T> child) {                  // (8)
        return new Subscriber<T>() {
            @Override
            public void onNext(T t) {
                child.onNext(t);
                pa.produced(1);
            }
            @Override
            public void onError(Throwable e) {
                pa.set(null);
                child.onError(e);
            }
            @Override
            public void onCompleted() {
                pa.set(null);
                child.onCompleted();
            }
            @Override
            public void setProducer(Producer producer) {
                pa.set(producer);
            }
        };
    }
}
~~~

这次扩展看起来不太优雅，怎么回事？

1. 我们创建了一个 `ProducerArbiter` 实例。
2. 在 `onNext` 中，我们通知 `ProducerArbiter` 已经生产了一个数据。
3. 在 `onError` 中，我们释放掉 producer，以避免内存泄漏。
4. 同样，在上游 `onCompleted` 事件到来时，我们也清除掉上游的 producer。
5. `Subscriber` 不应该通常也不能被复用，所以我们需要为第二个数据源创建一个新的 `Subscriber`。而且如果复用，将会导致无限的重订阅，最终导致 `StackOverflowError`。
6. 把新的 `Subscriber` 加入到 child 中，这样就能正确取消订阅了。我们订阅第二个 `Observable`，并且结束第一个 subscriber。
7. 我们捕获上游可能的 producer，并将其用在 `ProducerArbiter` 中。
8. 为了方便，我把创建第二个 `Subscriber` 的代码移到了一个辅助函数中。它的结构看起来和第一个 `Subscriber` 很类似，除了 `onCompleted` 稍有不同，我们结束 child，而不是再订阅其他的 observable。

## 总结

利用像 `ProducerArbiter` 这样的 producer，我们就可以动态切换 producer了，并且能保证请求数量的正确处理，因此从多个上游数据源的 producer 中获取正确的数据了。

然而，正如我在具体的解释中警告的那样，这个 producer-arbiter 的运行和 `onXXX` 事件是独立的，如果事件的发射没有保证串行发生，那就有可能出现不可预测的行为。但在本文的例子中，不会发生这样的问题，因为我们知道当我们在第一个 `Subscriber.onCompleted()` 中切换 producer 时，第一个 observable 不会再发出任何 `onNext` 和 `onError` 事件了，所以我们可以安全切换。

然而，在绝大多数诸如 `switchOnNext()` 高级操作符中，都存在并发的请求分发，所以如果想要它们能正常工作，我们需要一个更加高级的 producer-arbiter：**producer-observer-arbiter**。我将在本系列的下一篇文章中进行介绍。
