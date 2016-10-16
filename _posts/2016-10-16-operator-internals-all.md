---
layout: post
title: 深入理解 Operator：All，Any 和 Exists
tags:
    - Operator
---

原文 [Operator internals: All, Any, Exists](http://akarnokd.blogspot.com/2015/10/operator-internals-all.html){:target="_blank"}

## 介绍

`all` 这个操作符，会检查上游发出的所有数据是否都满足给定的条件（`predicate`），如果有任何一个数据不满足条件，就立即发出 `false` 然后结束，如果所有数据都满足条件（或者没有数据），就会发出 `true` 后结束。而 `any` 则和 `all` 在逻辑上相反，只要有一个数据满足条件，就立即发出 `true` 然后结束，如果所有数据都不满足条件（或者没有数据），就会发出 `false` 后结束。这两个操作符都满足 backpressure 的要求。

我们需要了解这两个操作符的以下特性/要求：

+ 由于发往下游的数据只有一个，所以我们不需要考虑如何向上游请求数据，直接向上游请求无限（`Long.MAX_VALUE`）个数据即可。这样做的好处是可能触发上游的快路径，进而减小运行开销。
+ 同样由于发往下游的数据只有一个，即便上游没有数据也会有一个数据发往下游，我们需要考虑来自下游的请求，只有在下游请求过之后才发出这个唯一的数据。

## 1.x 的实现

1.x 的实现非常直观，它向上游请求 `Long.MAX_VALUE` 个数据，利用 `SingleDelayedProducer` 来延迟对数据的发射，只有当下游请求了之后才发出。

由于 backpressure 在 1.x 中是可选的，所以我们无法在 `onNext` 中遇到不满足条件的数据之后立即发出 `false`，因为只有 `SingleDelayedProducer` 知道当前是否已经有了来自下游的请求（_我们要遵循 backpressure，但上游不一定遵循了，我们必须可靠地遵循 backpressure，所以我们必须经过 SingleDelayedProducer 中转_）。

## 2.x 的实现

2.x 的实现更长一些，因为我直接把 SingleDelayedProducer 的逻辑实现在了这个操作符中，这样可以减少内存分配。

backpressure 的要求没有变，但由于 `onSubscribe` 的调用是必须的，所以 `onNext` 到来时就说明下游一定已经请求过了数据。

如果在 `onNext` 中遇到了不满足条件的数据，就无需进行中转了，我们可以直接发往下游，因为上游发来了数据，就说明下游一定已经有了请求。但对于空的上游来说，我们还是需要进行中转的。这里的状态机和[前文](/AdvancedRxJava/2016/06/04/operator-concurrency-primitives-4/){:target="_blank"}中的状态机很类似。有一个值得注意的区别就在于，由于我们知道需要延迟发射的数据一定是 `true`，所以我们无需一个变量保存它的值了。

让我们看看 `AllSubscriber#onNext()`：

~~~ java
@Override
public void onNext(T t) {
    if (done) {                             // (1)
        return;
    }
    boolean b;
    try {
         b = predicate.test(t);
    } catch (Throwable e) {                 // (2)
         lazySet(HAS_REQUEST_HAS_VALUE);
         done = true;
         s.cancel();
         actual.onError(e);
         return;
    }
    if (!b) {
        lazySet(HAS_REQUEST_HAS_VALUE);     // (3)
        done = true;
        s.cancel();
        actual.onNext(false);
        actual.onComplete();
    }
}
~~~

1. 取消订阅在 Reactive-Streams 和 RxJava 1.x 中都只是尽力而为，我们的逻辑不能完全依赖取消订阅。利用 `done` 标记，我们会在结束/取消之后，丢弃所有数据。
2. 检查数据的代码可能出错，出错之后我们就把状态设置为终结状态 `HAS_REQUEST_HAS_VALUE`，这会让 `request()` 函数不请求新的数据。同时我们设置 done 标记，并且取消上游。
3. 如果数据不满足条件，我们可以直接向下游发出 false，并且取消上游。同样，状态机也会进入终结状态。

## 总结

`all` 和 `any` 操作符很简单，我觉得按从简单到复杂排名可以进入前 20%，但如果上游是空的，天真的实现方式可能会违背 backpressure 的要求，发出下游没有请求过的数据，因此我们需要 `SingleDelayedProducer` 来填补这个空缺。
