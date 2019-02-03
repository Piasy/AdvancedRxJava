---
layout: post
title: Operator 并发原语： producers（一），RangeProducer
tags:
    - Operator
    - Producer
---

原文 [Operator concurrency primitives: producers (part 1)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_12.html){:target="_blank"}

## 介绍
**Backpressure** 从 RxJava 1.0 正式引入，主要用于进行流量控制，以及把各种操作符无限长度的缓冲队列控制在有限长度内。操作符 backpressure 的实现基于协程（coroutine），实际上我对协程了解甚少。我们可以通过对多种 producer 的实现和使用来更加直观地理解操作符 backpressure 的实现。

如果你对 [reactive-streams-jvm](https://github.com/reactive-streams/reactive-streams-jvm){:target="_blank"} 规范比较熟悉，你就会发现其中的 `Subscription` 和 RxJava 的 `Producer` 比较相似。由于本文关注于 RxJava 1.0，所以我将主要从 RxJava 的角度出发。

`Producer` 是 RxJava 中的一个接口，它是传递 backpressure 相关信息的主要方式，通过它我们可以在整条操作符链条中 _请求 n 个元素_。它唯一的方法就是 `request(long n)`，在 subscribe 建立操作符链条的过程中，上游的操作符会通过 `setProducer` 接口把它设置给 `Subscriber`。

在我们欢庆 `Producer` 成为一个 `@FunctionalInterface` 之前，我不得不很遗憾地指出，如果不考虑**获取状态（state）**以及支持**取消（cancellation）**和获取**已生产数据（values-produced）**的方法，我们很难设计出有价值的 producer。

有人可能会把 `Producer` 作为实现通往上游的 `long` 数据流的一种方式，和现有的通往下游的 `onXXX` 事件流相反。但我把这两种流都当成通往下游的事件流，这有助于我们理解两者的并发行为，绝大部分的高级操作符都通过这样的并发方式来保证正确性。

我在本文中，我不会完全从概念上出发，而是通过实现一个类似于 `Observable.range()` 这样发射特定数量数字的的 producer ，来讲解 producer 的概念。

## The range producer
当我初识 RxJava 中的 producer 时，我很难理解 Netflix 实现的几个版本，但看到 `range()` 和 `from()` 操作符背后的 producer 时，我瞬间就懂了。

简单来说，`Observable.range()` 工厂方法中的 producer 会在下游请求 `n` 个数据的时候，生产 `n` 个（递增的）数据。

~~~ java
Observable<Integer> range = Observable.create(child -> {   // (1)
    int[] index = new int[] { 0 };                         // (2)
    Producer p = n -> {                                    // (3)
       int j = index[0];
       for (int i = 0; i < n; i++) {                       // (4)
           child.onNext(j);
           j++;
           if (j == 100) {                                 // (5)
              child.onCompleted();
              return;
           }
       }
       index[0] = j;                                       // (6)
    };
 
    child.setProducer(p);                                  // (7)
});
~~~

_警告：这个 producer 的例子中缺少几个必要的特性，我将在后续的例子中加上，所以千万不要在你的代码中实现和运行这样的 producer。_

让我们逐步分析这个简单的 range Observable：

1. 我们利用 `create()` 工厂方法创建了一个 Observable，我们传入了一个 `OnSubscribe` lambda 表达式，通过它来引用之后 subscribe 的 `child` `Subscriber`。
2. 我很喜欢 lambda 表达式。我们需要保存序列结束的位置，以便我们后续继续生成数据。
3. 我们的 Producer 也是用 lambda 实现的（实际场景很可能不是这样），当它被调用时，它将读取上次停止的数据位置。
4. Producer 在循环中依次发出 `n - 1` 个数据，值从 `j` 开始依次递增。
5. 我们会在循环中检查是否总共已发射了 100 个数据，是则终止 `child`。
6. 如果总共没有发射超过 100 个数据，我们就会在退出循环时把 `j` 的值保存到 `index` 中，用于后续的 `request()` 调用继续发射。
7. 最后，我们把 `p` 设置给了 `Subscriber`。

如果我们在同步场景中使用上述 `range` Observable，它能很好地工作：

~~~ java
range.subscribe(
    System.out::println, 
    Throwable::printStackTrace, 
    () -> System.out.println("Done"));
 
TestSubscriber<Integer> ts = new TestSubscriber<>();
ts.requestMore(0);
 
range.subscribe(ts);
 
ts.requestMore(25);
ts.getOnNextEvents().forEach(System.out::println);
 
ts.requestMore(10);
ts.getOnNextEvents().forEach(System.out::println);
 
ts.requestMore(65);
ts.getOnNextEvents().forEach(System.out::println);
~~~

然而如果我们加上一系列复杂的操作符（包括 `observeOn`）之后，将会出现奇怪的错误：它可能会发射出所有的数据，也可能会打印重复的数据，甚至导致程序挂起（hang）或者抛出 `MissingBackpressureException`。

出现上述未定义行为的原因，是由于我们的 `Producer` 不是 _线程安全（thread-safe）_ 的，也不是 _可重入（reentrant-safe）_ 的。

对 `Producer.request()` 方法的**线程安全要求**来自于它可能会被多个操作符异步调用，如果不满足这一要求，Producer 的内部状态就会遭到破坏。

而**可重入要求**来自于可能会有一些操作符会在某次 `request()` 尚未执行完毕时，再次（在另一个线程）调用 `request()`，如果不满足这一要求，就会导致 Producer 发射重复的数据，因为此时 `j` 的值还没有保存到 `index` 中。

（奇怪的是，这两个要求并未在 reactive-streams-jvm 规范中强制提出，尽管我为此展开过讨论，以及依据规范实现了一套 API 以进行实验。尽管有些操作符只需要保证可重入就不会出现问题，但我认为非直观的操作符实现中，我们应该持保守态度，并且严格遵守上述两个要求。）

## 解决 range 中的安全性问题
现在我们就可以在 producer 的 `request` 方法中运用在[之前的文章](/AdvancedRxJava/2016/05/13/operator-concurrency-primitives-2/index.html){:target="_blank"}中讲解的串行访问原则了。由于请求数据的个数也是一个数字，所以我们可以把它和 `wip` 合并起来以节省内存。同时我们可以直接继承 `AtomicLong` 并实现 `Producer` 以进一步节约内存（每个 producer 实例至少减少 24 字节）。

由于 producer 的逻辑稍微有点复杂，所以我先讲解一下 `RangeProducer` 的类结构：

~~~ java
public final class RangeProducer 
extends AtomicLong implements Producer {
    private static final long serialVersionUID = 1;
 
    final Subscriber<? super Integer> child;                  // (1)
    int index;                                                // (2)
    int remaining;                                            // (3)
     
    public RangeProducer(
            Subscriber<? super Integer> child, 
            int start, int count) {
        if (count <= 0) {
            throw new IllegalArgumentException();             // (4)
        }
        this.child = child;
        this.index = start;
        this.remaining = count;
    }
 
    @Override
    public void request(long n) {
        // the logic comes here
    }
}
~~~

类结构很简单：

1. 由于我们的 producer 生产的数据需要被消费，所以我们需要持有 `child` 的引用，这一点在之前的例子中并没有这么直接。
2. `index` 变量记录下一个生产的数据的值。
3. `remaining` 变量记录还能生产多少个数据，每次发射一个数据之后就会将其递减，减到零之后就停止发射。
4. 非正的 `count` 参数是非法的，如果我们在 `range` 的 `OnSubscribe` 中预先判断需要生产的数据量，当 `count` 为零时，我们可以直接调用 `onCompleted()` 方法，避免 producer 对象的分配，节约内存。

首先，我们需要执行处理请求的操作：

~~~ java
// ...
@Override
public void request(long n) {
    if (n < 0) { 
        throw new IllegalArgumentException();           // (1)
    }
    if (n == 0) {
        return;                                         // (2)
    }
    long r;
    for (;;) {
        r = get();                                      // (3)
        long u = r + n;                                 // (4)
     
        if (u < 0) {
            u = Long.MAX_VALUE;                         // (5)
        }
     
        if (compareAndSet(r, u)) {                      // (6)
            break;
        }
    }
    // ... will be continued
~~~

1. 我们先检查请求参数是否为负数，请求负数肯定是 bug，所以我们会抛出 `IllegalArgumentException`。
2. 请求零个数据我们无需进行任何实际操作，但我们需要特殊考虑这种情况，以避免其触发后续的漏循环。
3. 我们需要记录总生产的数据量，将其保存到 `AtomicLong` 中。我们需要在循环中进行 CAS 操作，因为我们需要保证总数量不超过 `Long.MAX_VALUE`，并且保证这一操作的原子性。
4. 把新请求的数据量加到总数据量中。
5. 由于加法过程可能发生溢出，我们保证总数量不超过 `Long.MAX_VALUE`，我们可以把它看做无穷大。
6. 我们尝试为计数赋予新的值，一旦成功我们就退出循环。因为可能有并发的线程会改变计数的值，这种情况下我们就需要重试。

实际上上面的循环在 RxJava 中使用非常广泛，RxJava 中有一个工具方法 `BackpressureUtils.getAndAddRequest()` 封装了上面的这个循环。

~~~ java
        if (r != 0) {
            return;                             // (1)
        }
 
        r = n;                                  // (2)
 
        for (;;) {
            int i = index;                      // (3)
            int k = remaining;
            int e = 0;
 
            while (r > 0 && k > 0) {            // (4)
                child.onNext(i);
                k--;
                if (k == 0) {                   // (5)
                    child.onCompleted();
                    return;
                }
                i++;                            // (6)
                e++;
                r--;
            }
 
            remaining = k;                      // (7)
            index = i;
 
            r = addAndGet(-e);                  // (8)
 
            if (r == 0) {
                break;                          // (9)
            }
        }
    } // end of method
}
~~~

我承认，`request` 方法的后半部分看起来很复杂，但理解它对于理解 producer 的工作原理非常有帮助：

1. 一旦我们退出了 CAS 循环，如果当前正在生产的数量（_还未生产完，仍需要继续生产的数量_） `r` 等于 0，那我们就开始生产数据。而如果不为 0，就说明当前已经有线程正在生产了。
2. 首先我们假设需要生产的数据量是 `n`（2），当然并发的调用可能会改变 `AtomicLong` 的值，但我们每次在（8）处都会重新获取新的值，从而节省了不必要的内存屏障（memory barrier）。
3. 我们把 producer 的状态保存到局部变量中，避免后续操作时由于下游的原子操作重新读取。
4. 接下来我们进入漏循环，我们会在可以继续生产数据（不超过 `r`）的前提下，在循环中生产下游请求的数量（`k`）。一旦生产了一个数据，我们立即递减仍需生产的数据计数。
5. 如果所有的数据都生产完了，我们就立即调用 `onCompleted()` 并退出，无需再修改任何变量。这种情况下我们并没有把待生产数据计数减到零，所以它仍处于生产状态，所以后续的 `request()` 都不会有任何效果（_也不应该有任何效果，因为它已经结束了_）。
6. 否则我们就递增 `index`，递增已生产数据计数，递减缓存的请求数量。
7. 一旦循环结束，我们把缓存的剩余可生产的数据个数以及 `index` 写回成员变量中。需要注意的是，它们的新值可见性是由 `compareAndSet()` 和 `getAndAdd()` 方法对保证的：对它们的修改，将在下次进入（2）的时候被其他线程看见。
8. 这一步是处理请求的收尾部分，我们利用原子操作减去本次发射的数据量，它和队列漏例子中用的 `decrementAndGet()` 类似，要么我们会“安全地”退出循环（_后续线程会继续循环_），要么我们会重新继续循环（其他的线程有可能会并发调用了 `request()`）。
9. 当 `r` 变为零时，表明我们当前所有的请求都已经处理完毕了，下游的订阅者/操作符不再需要新的数据，所以我们可以退出循环了。如果有并发的线程在（8）之后调用了 `request()`，它将会把待处理请求数从 0 增加到参数 `n`，这种情况下，它将进入循环开始生产数据。

在结束本文之前，还有很重要的一点值得一提：取消订阅（unsubscription）。下游可以在任何时候从任何线程取消订阅，但我们实现的 producer 会继续傻傻地生产数据。为了避免浪费 CPU 的计算资源，我们需要在内循环中通过 `isUnsubscribed()` 检查下游订阅者是否仍接收数据。

~~~ java
// ...
if (child.isUnsubscribed()) {              // (1)
    return;
}
while (r > 0 && k > 0) {
    child.onNext(i);
    if (child.isUnsubscribed()) {          // (2)
        return;
    }
    k--;
    if (k == 0) {
        child.onCompleted();
        return;
    }
// ... the rest is the same
~~~

如果你想要，你可以在任何位置调用 `isUnsubscribed()` 进行检查，但通常来说只需要在循环之前进行检查（1）以及在每次发射新数据之后进行检查（2）。前者可以彻底避免不必要的循环，后者可以在每次发射数据之后避免此后不必要的发射以及 `onCompleted()`。

值得注意的是，RxJava 的实现中关于取消订阅之后的结束事件行为并不一致，绝大部分操作符都能很好地工作，但可能有少部分存在一些副作用。但是规范允许取消订阅之后仍继续发射事件或者结束信号，因为规范就是把取消订阅定义为“尽力而为”（best effort）的。

_如果你发现有些操作符在响应取消订阅之后的事件行为很奇怪，请立即在 [RxJava issue 列表](https://github.com/ReactiveX/RxJava/issues){:target="_blank"}中提交 issue。_

## 总结
在本文中，我逐步展示了如何编写一个简单地 producer，以及它们的工作原理。我也解释了这些简单的 producer 需要避免的陷阱，以满足 RxJava 的需求。在本文结尾，我贴出功能完整、可运行的 Range 操作符的代码（[gist](https://gist.github.com/akarnokd/280a5304fa8d229b1e16){:target="_blank"}）。

~~~ java
public final class RxRange 
implements OnSubscribe<Integer> {
    final int start;
    final int count;
    public RxRange(int start, int count) {
        if (count < 0) {
            throw new IllegalArgumentException();
        }
        this.start = start;
        this.count = count;
    }
    @Override
    public void call(Subscriber t) {
        if (count == 0) {
            t.onCompleted();
            return;
        }
        RangeProducer p = new RangeProducer(t, start, count);
        t.setProducer(p);
    }
     
    public Observable<integer> toObservable() {
        return Observable.create(this);
    }
     
    static final class RangeProducer 
    extends AtomicLong implements Producer {
        /** */
        private static final long serialVersionUID = 
            5318571951669533517L;
        final Subscriber<? super Integer> child;
        int index;
        int remaining;
        public RangeProducer(
                Subscriber<? super Integer> child,
                int start, int count) {
            this.child = child;
            this.index = start;
            this.remaining = count;
        }
        @Override
        public void request(long n) {
            if (n < 0) {
                throw new IllegalArgumentException();
            }
            if (n == 0) {
                return;
            }
            if (BackpressureUtils.getAndAddRequest(this, n) != 0) {
                return;
            }
            long r = n;
            for (;;) {
                if (child.isUnsubscribed()) {
                    return;
                }
                int i = index;
                int k = remaining;
                int e = 0;
                 
                while (r > 0 && k > 0) {
                    child.onNext(i);
                    if (child.isUnsubscribed()) {
                        return;
                    }
                    k--;
                    if (k == 0) {
                        child.onCompleted();
                        return;
                    }
                    e++;
                    i++;
                    r--;
                }
                index = i;
                remaining = k;
                 
                r = addAndGet(-e);
                 
                if (r == 0) {
                    return;
                }
            }
        }
    }
     
    public static void main(String[] args) {
        Observable<Integer> range = 
            new RxRange(1, 10).toObservable();
         
        range.take(5).subscribe(
            System.out::println,
            Throwable::printStackTrace,
            () -> System.out.println("Done")
        );
    }
}
~~~

在下一篇博客中，我将继续继续讨论更高级的 producer，它们在关注 backpressure 的操作符中很常见：**single-producers** 和 **single-delayed-producers**。
