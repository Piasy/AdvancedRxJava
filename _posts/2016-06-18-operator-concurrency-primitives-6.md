---
layout: post
title: Operator 并发原语： producers（四），RangeProducer 优化
tags:
    - Operator
    - Producer
---

原文 [Operator concurrency primitives: producers (part 4)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_13.html){:target="_blank"}

## 介绍

在实现了[相对复杂的 producer](/AdvancedRxJava/2016/06/11/operator-concurrency-primitives-5/){:target="_blank"} 之后，现在是时候关注简单一点的内容了。在本文中，我将对最初介绍的 `RangeProducer` 进行一次优化：在无限请求时增加一个发射快路径。

在 RxJava 中，如果第一次就请求 `Long.MAX_VALUE` 等同于请求无限的数据，并且会触发很多发射快路径，就像支持 backpressure 之前的远古时代那样。在这种情况下，我们无需响应请求并生产数据了（只需处理好取消订阅即可）。

## `RangeProducer` 的快路径

我只列出 `request()` 方法的代码，因为其他部分的代码完全没有变化：

~~~ java
// ... same as before
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
    if (n == Long.MAX_VALUE) {                                // (1)
        if (child.isUnsubscribed()) {
            return;
        }
        int i = index;                                        // (2)
        int k = remaining;
        while (k != 0) {
            child.onNext(i);
            if (child.isUnsubscribed()) {                     // (3)
                return;
            }
            i++;                                              // (4)
            k--;
        }
        if (!child.isUnsubscribed()) {
            child.onCompleted();
        }
        return;                                               // (5)
    }
 
    long r = n;
    for (;;) {
// ... same as before
~~~

快路径的工作原理如下：

1. 如果我们成功把计数从 0 增加到 n，并且 n 为 `Long.MAX_VALUE`，我们就进入了快路径，如果 n 小于 `Long.MAX_VALUE`，我们将执行慢路径。
2. 我们把 producer 的状态读取到局部变量中。注意，如果之前在慢路径中发射过数据，那我们读取到的值将反映出我们继续发射的位置。如果当前这次无限的请求得到了发射的权利（_当然得到了，因为现在我们已经进入了快路径_）。
3. 检查 child 是否已经取消了订阅。
4. 我们递增 `i`，递减 `k`。
5. 在所有的数据以及结束事件发射完毕之后，我们就直接退出执行，而不再调整内部的请求计数。这确保了结束之后的请求既不会进入快路径，也不会进入慢路径，因为 `BackpressureUtils.getAndAddRequest` 永远不会成功。

注意，小量请求后接着一个无限请求这种情况在 RxJava 中不会发生。操作符要么开启了 backpressure，要么没有开启 backpressure，所以我们无需担心，如果无限请求在慢路径循环中和 `r = addAndGet(-e);` 之间到来并且可能把请求计数递减到 `Long.MAX_VALUE` 之下，而导致我们被陷在慢路径中。

## 实现一个基于数组的 producer

RxJava 的 `from()` 操作符支持传入一个 `T` 类型的数组，但在其内部实现中，这个数组会在 producer 中被转化为一个列表并进行遍历。这种方式看起来不必要，既然我们拿到的是一个已知长度的数组，那我们就无需 `Iterator` 而是直接利用下标进行遍历了（你可能会认为 JIT 会对此进行优化，使得 `Iterator` 在栈上进行分配，但 `onNext()` 中的代码有可能会阻止此项优化）。另外，由于 `from()` 不支持基本类型的数组，所以你可能需要自行编写一个支持此类型的操作符。

`RangeProducer` 的结构是实现这个功能的一个不错的选择：我们可以用 `index` 来记录当前遍历到数组的下标，然后把它和数组长度进行对比以决定何时退出。

~~~ java
public final class ArrayProducer 
extends AtomicLong implements Producer {
    /** */
    private static final long serialVersionUID = 1L;
    final Subscriber child;
    final int[] array;                                        // (1)
    int index;
    public ArrayProducer(Subscriber child, 
            int[] array) {
        this.child = child;
        this.array = array;
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
        final int[] a = this.array;
        final int k = a.length;                               // (2)
        if (n == Long.MAX_VALUE) {
            if (child.isUnsubscribed()) {
                return;
            }
            int i = index;
            while (i != k) {                                  // (3)
                child.onNext(a[i]);
                if (child.isUnsubscribed()) {
                    return;
                }
                i++;
            }
            if (!child.isUnsubscribed()) {
                child.onCompleted();
            }
            return;
        }
        long r = n;
        for (;;) {
            if (child.isUnsubscribed()) {
                return;
            }
            int i = index;
            int e = 0;
             
            while (r > 0 && i != k) {
                child.onNext(a[i]);
                if (child.isUnsubscribed()) {
                    return;
                }
                i++;
                if (i == k) {                               // (4)
                    child.onCompleted();
                    return;
                }
                e++;
                r--;
            }
            index = i;
             
            r = addAndGet(-e);
             
            if (r == 0) {
                return;
            }
        }
    }
}
 
int[] array = new int[200];
Observable<Integer> source = Observable.create(child -> {
    if (array.length == 0) {
        child.onCompleted();
        return;
    }
    ArrayProducer ap = new ArrayProducer(child, array);
    child.setProducer(ap);
});
source.subscribe(System.out::println);
~~~

1. 除了 `index` 之外，我们还需要 `array` 来保存待发射的数组，我们无需 `remaining` 了，因为 `index` 最多递增到数组的长度。
2. 结束运行的条件是局部变量 `i` 递增到 `k`（数组长度）。注意我们无需递减 `k`。
3. 在快路径中，在 `i` 递增到数组长度之前我们都进行循环。
4. 在慢路径中，每次递增 `i` 之后，我们立即检查是否已经抵达了数组的末尾，如果抵达末尾就发出 `onCompleted()`。注意，慢路径中不支持空数组。

## 总结

在本文中，我展示了如何为简单如 `RangeProducer` 的 producer 增加一个快路径，并且如何把它转变为一个支持基本类型数组的 producer，避免额外的 `Iterator` 分配和遍历开销。

到目前为止，我介绍了众多的 producer，包括确切知道应该发射多少数据的 producer，以及不知道或者不关心发射量的 producer。然而，存在一些需要处理来自多种 producer 的多个数据源的操作符，它们还需要处理得 child 只需要处理一种数据源。在下一篇关于 producer 的文章中，我将介绍一种我称之为 **producer-arbiter** 的 producer，它能在保证 backpressure 的前提下支持不同 producer 之间进行切换。
