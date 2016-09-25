---
layout: post
title: Reactive-Streams API（二）：SingleSubscription，SingleDelayedSubscription 和 RangeSubscription
tags:
    - Reactive Stream
---

原文 [The Reactive-Streams API (part 2)](http://akarnokd.blogspot.com/2015/06/the-reactive-streams-api-part-2.html){:target="_blank"}

## 介绍

在本文中，我将把我们以前的 `SingleProducer` 和 `SingleDelayedProducer` 移植到基于 reactive-streams 的 `Subscription`。

首先，很多人可能认为这个转换过程很麻烦，但幸运的是，如果我们已经想清楚了在 `rx.Producer` 中如何实现 `request()`，那我们基本上就已经完成了 75% 了。剩下的 25% 来自于我们要把 `rx.Subscriber.isUnsubscribed()` 中的逻辑转移到 `request()` 中，因为 `org.reactivestreams.Subscriber` 中没有 `isUnsubscribed()`（其他资源管理类的接口都没有这个方法了）。

## `SingleSubscription`

由于 `SingleSubscription` 本身并不复杂，这里我就直接一步到位：

~~~ java
import org.reactivestreams.*;
 
public final class SingleSubscription<T> 
extends AtomicBoolean implements Subscription {
    private static final long serialVersionUID = 1L;
     
    final T value;                                       // (1)
    final Subscriber<? super T> child;
    volatile boolean cancelled;                          // (2)
     
    public SingleSubscription(T value, 
            Subscriber<? super T> child) {               // (3)
        this.value = Objects.requireNonNull(value);
        this.child = Objects.requireNonNull(child);
    }
    @Override
    public void request(long n) {
        if (n <= 0) {
            throw new IllegalArgumentException(
                "n > 0 required");                       // (4)
        }
        if (compareAndSet(false, true)) {
            if (!cancelled) {                            // (5)
                child.onNext(value);
                if (!cancelled) {
                    child.onComplete();
                }
            }
        }
    }
    @Override
    public void cancel() {
        cancelled = true;                                // (6)
    }
}
~~~

就是这样！这里我向大家展示了把 `Producer` 的实现迁移到 reactive-streams `Subscription` 并不需要太多的工作。但这里还是有几点值得一提：

1. 我们需要在成员变量中保存将要发出的值，以及下游 subscriber。
2. 然而由于 RS 中没有了 `isUnsubscribed()`，而且取消订阅也变成了 `cancel()`，所以我们需要在一个 `volatile` 变量中记录是否已经取消订阅。如果你还记得的话，我说过我们无法预知 `request()` 和 `cancel()` 的调用情况，所以我们需要保证它们的线程安全性。
3. 由于 RS 不允许 `null`，我们在构造函数中就检查错误。
4. 我的“Let them throw!”哲学告诉我们非正请求数量是编程错误，我们应该抛出 `IllegalArgumentException`。
5. 由于没有 `child.isUnsubscribed()` 函数了，我们只能检查我们的 `cancelled` 变量。
6. 为了保证取消的幂等性，我们只是安全的更改 `cancelled` 变量。

## `SingleDelayedSubscription`

`SingleSubscription` 都这么简单了，`SingleDelayedSubscription` 又能复杂到哪里去呢？

~~~ java
public final class SingleDelayedSubscription<T> 
extends AtomicInteger implements Subscription {
    private static final long serialVersionUID = -1L;
     
    T value;
    final Subscriber<? super T> child;
     
    static final int CANCELLED = -1;                           // (1)
    static final int NO_VALUE_NO_REQUEST = 0;
    static final int NO_VALUE_HAS_REQUEST = 1;
    static final int HAS_VALUE_NO_REQUEST = 2;
    static final int HAS_VALUE_HAS_REQUEST = 3;
     
    public SingleDelayedSubscription(Subscriber<? super T> child) {
        this.child = Objects.requireNonNull(child);
    }
    @Override
    public void request(long n) {
        if (n <= 0) {
            throw new IllegalArgumentException("n > 0 required");
        }
        for (;;) {
            int s = get();
            if (s == NO_VALUE_HAS_REQUEST
                    || s == HAS_VALUE_HAS_REQUEST
                    || s == CANCELLED) {                       // (2)
                return;
            } else if (s == NO_VALUE_NO_REQUEST) {
                if (!compareAndSet(s, NO_VALUE_HAS_REQUEST)) {
                    continue;
                }
            } else if (s == HAS_VALUE_NO_REQUEST) {
                if (compareAndSet(s, HAS_VALUE_HAS_REQUEST)) {
                    T v = value;
                    value = null;
                    child.onNext(v);
                    if (get() != CANCELLED) {                  // (3)
                        child.onComplete();
                    }
                }
            }
            return;
        }
    }
     
    public void setValue(T value) {
       Objects.requireNonNull(value);
       for (;;) {
           int s = get();
           if (s == HAS_VALUE_NO_REQUEST
                   || s == HAS_VALUE_HAS_REQUEST
                   || s == CANCELLED) {                        // (4)
               return;
           } else if (s == NO_VALUE_NO_REQUEST) {
               this.value = value;
               if (!compareAndSet(s, HAS_VALUE_NO_REQUEST)) {
                   continue;
               }
           } else if (s == NO_VALUE_HAS_REQUEST) {
               if (compareAndSet(s, HAS_VALUE_HAS_REQUEST)) {
                   child.onNext(value);
                   if (get() != CANCELLED) {                   // (5)
                       child.onComplete();
                   }
               }
           }
           return;
       }
    }
 
    @Override
    public void cancel() {
        int state = get();
        if (state != CANCELLED) {                              // (6)
            state = getAndSet(CANCELLED);
            if (state != CANCELLED) {
                value = null;
            }
        }
    }
}
~~~

看起来和原来的状态机非常类似，但是多了一个 `CANCELLED` 状态，我们无需在 `onNext` 之前检查状态不为 `CANCELLED`，因为 CAS 操作隐含了这一条件，但我们应该在 `onComplete()` 之前进行检查。

为什么我们不使用一个 `volatile boolean` 来记录是否取消呢？其实完全可以。这种选择仅仅是出于个人偏好：增加一个成员变量，或者是扩展一个新状态。我主要是想要展示一下后者怎么实现。

## `RangeSubscription`

我并不打算在这里把以前所有的 `Producer` 都改写为 `Subscription`，但我这里还想展示一个包括取消状态的状态机例子：

~~~ java
public final class RangeSubscription 
extends AtomicLong implements Subscription {
    private static final long serialVersionUID = 1L;
     
    final Subscriber<? super Integer> child;
    int index;
    final int max;
     
    static final long CANCELLED = Long.MIN_VALUE;          // (1)
     
    public RangeSubscription(
            Subscriber<? super Integer> child, 
            int start, int count) {
        this.child = Objects.requireNonNull(child);
        this.index = start;
        this.max = start + count;
    }
    @Override
    public void request(long n) {
        if (n <= 0) {
            throw new IllegalArgumentException(
                "n > required");
        }
        long r;
        for (;;) {
            r = get();
            if (r == CANCELLED) {                          // (2)
                return;
            }
            long u = r + n;
            if (u < 0) {
                u = Long.MAX_VALUE;
            }
            if (compareAndSet(r, u)) {
                break;
            }
        }
        if (r != 0L) {                                     // (p1)
            return;
        }
        for (;;) {
            r = get();
            if (r == CANCELLED) {                          // (3)
                 return;
            }
            int i = index;
            int m = max;
            long e = 0;
            while (r > 0L && i < m) {                      // (p2)
                child.onNext(i);
                if (get() == CANCELLED) {                  // (4)
                    return;
                }
                i++;
                if (i == m) {
                    child.onComplete();
                    return;
                }
                r--;
                e++;
            }
            index = i;
            if (e != 0) {
                for (;;) {
                    r = get();
                    if (r == CANCELLED) {                  // (5)
                        return;
                    }
                    long u = r - e;
                    if (u < 0) {
                        throw new IllegalStateException(
                                "more produced than requested!");
                    }
                    if (compareAndSet(r, u)) {
                        break;
                    }
                }
            }
            if (r <= 0L) {                                 // (p3)
                break;
            }
        }
    }
    @Override
    public void cancel() {
        if (get() != CANCELLED) {                          // (6)
            getAndSet(CANCELLED);
        }
    }
}
~~~

为了简洁起见，我省略了快速路径的逻辑。剩下的部分和原来的 `RangeProducer` 类似，但是取消状态被合并进了计数状态中，我们几乎需要在所有的地方（1~5）重新读出计数并和 `CANCELLED` 对比。注意发射计数再也不能用 `getAndAdd()` 了，直接递增可能会覆盖掉 `CANCELLED` 状态。最后在取消时使用 `getAndSet()` 可以保证幂等性。

_译者注：这一段代码还是很复杂的，即便我之前翻译过 `RangeProducer` 的实现，看起来依然需要一些思考，所以这里进行一些分析（对应于上面的 p 标号）：_

1. 当我们成功更新请求计数之后，只有从 0 开始增加请求计数的线程可以进入后面的发射循环。发射过程会递减请求计数，当请求处理完毕之后（请求计数重新变为 0），下次的请求调用才有可能进入发射循环。
2. 这里发射循环有两个条件，一是有未处理的请求（`r > 0`），而是发射数据没有超出范围（`i < m`），这两者很可能是不同的。此外，如果发射数据已经超出了范围，而且请求计数也递减为 0 了，那后续的请求仍然能通过（p1）的检查，但不会进入发射循环，因为通不过 `i < m` 的检查。
3. 这里是请求处理完毕，但发射数据没有超出范围的退出路径。

## 总结

在本文中，我展示了两种把 `rx.Producer` 改写为 RS `Subscription` 的方式（boolean 成员或者新状态），它们都能保证取消行为的正确性。对它们进行取舍需要进行权衡：boolean 成员会增加对象大小，新状态会增加算法复杂性。

下一篇文章中，我将介绍如何处理 RS 中缺失的另一个 `rx.Subscriber` 特性：用 `add(rx.Subscriber)` 把资源和下游 subscriber 结合起来。
