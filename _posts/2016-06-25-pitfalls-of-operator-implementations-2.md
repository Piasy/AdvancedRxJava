---
layout: post
title: 实现操作符时的一些陷阱（二）
tags:
    - Operator
    - Pitfall
---

原文 [Pitfalls of operator implementations (part 2)](http://akarnokd.blogspot.com/2015/05/pitfalls-of-operator-implementations_14.html){:target="_blank"}

## 介绍

本文中我将暂停对 producer 的讲解，继续回到实现操作符的陷阱这个话题，而且还会提到使用特定（序列）的 RxJava 操作符时的一些陷阱。

## 6，不等请求就直接发射（Emitting without request）

假设你要实现一个操作符，它会忽略上游发出的任何数据，并在上游结束时发出一个固定的值：

~~~ java
Operator<Integer, Integer> ignoreAllAndJust = child -> {
    Subscriber<Integer> parent = new Subscriber<Integer>() {
        @Override
        public void onNext(Integer value) {
            // ignored
        }
        @Override
        public void onError(Throwable e) {
            child.onError(e);
        }
        @Override
        public void onCompleted() {
            child.onNext(1);
            child.onCompleted();
        }
    };
    child.add(parent);
    return parent;
};
~~~

上面的这个操作符依赖于两个前提：`Subscriber` 的默认行为就是发出请求；下游一定会在上游结束之前至少发出一次请求。然而，即便你的测例通过了，这个操作符依然违反了 backpressure 的要求：`onCompleted()` 函数无条件的发出了一个数据，没有检查下游是否发出过请求。这个问题会在这样的场景下暴露出来：如果你有一个 hot `Observable` 或者不考虑 backpressure 的 `Observable`，而你又需要和 [reactive-streams](https://github.com/reactive-streams/reactive-streams-jvm){:target="_blank"} 兼容的下游进行交互，那么下游的 `Subscriber` 就会收到 `onError` 了，因为你的行为违反了 reactive-streams 规则的 §1.1 节。

既然我们现在已经了解了很多 producer，修复这个问题非常简单：

~~~ java
// ... same as before
@Override
public void onCompleted() {
    child.setProducer(new SingleProducer(child, 1));
}
// ... same as before
~~~

我们在 [produer（二）](/AdvancedRxJava/2016/06/04/operator-concurrency-primitives-4/){:target="_blank"} 中介绍过 `SingleProducer`，现在它是最合适的选择。

但是我想介绍另外一种解决方案，这种方案和 RxJava 2.0 以及 reactive-streams 兼容的操作符相关：

~~~ java
Operator<Integer, Integer> ignoreAllAndJust = child -> {
    SingleDelayedProducer<Integer> sdp = 
        new SingleDelayedProducer<>(child);
    Subscriber<Integer> parent = new Subscriber<Integer>() {
        @Override
        public void onNext(Integer value) {
            // ignored
        }
        @Override
        public void onError(Throwable e) {
            child.onError(e);
        }
        @Override
        public void onCompleted() {
            sdp.set(1);
        }
    };
    child.add(parent);
    child.setProducer(sdp);
    return parent;
};
~~~

这种方案功能上是一样的，尽管相较于 RxJava 1.x 稍显冗长。之所以需要这样，是因为操作符的 Subscriber 无法脱离 Producer 而单独存在。而这是因为 Producer 语义上来说也是一种 Subscription，而且它为 Subscriber 提供了从上游取消订阅的唯一途径。延迟设置 producer 会延迟可能的取消订阅。

## 7，操作符中的共享状态（Shared state in the operator）

你可能认为 `ignoreAllAndJust` 很傻也没什么用处，但如果我们把它改成一个在接收到上游数据时进行计数，并在上游结束时发出这个计数，那它就变得有点用处了。假设我们的编译环境是 Java 6，不能用 lambda 表达式：

~~~ java
public final class CounterOp<T 
implements Operator<Integer, T> {
    int count;                                              // (1)
    @Override
    public Subscriber call(Subscriber child) {
        Subscriber<T> parent = new Subscriber<T>() {
            @Override
            public void onNext(T t) {
                count++;
            }
            @Override
            public void onError(Throwable e) {
                child.onError(e);
            }
            @Override
            public void onCompleted() {
                child.setProducer(
                    new SingleProducer<Integer>(child, count));
            }
        };
        child.add(parent);
        return parent;
    }
}
 
Observable<Integer> count = Observable.just(1)
    .lift(new CounterOp<Integer>());                        // (2)
         
count.subscribe(System.out::println);
count.subscribe(System.out::println);
count.subscribe(System.out::println);
~~~

我们已经吸取了上一条的教训，正确实现了 `onCompleted()` 方法，然而如果运行上面的代码，我们会发现打印的结果是 `1`，`2` 和 `3`！显然 `just(1)` 的计数应该始终是 `1`，无论我们对它计数多少次。

问题就出在（1）处，我们在所有的订阅者中共享了 `count` 变量。第一个订阅者会把它增加到 `1`，第二个订阅者会把它增加到 `2`，以此类推，由于（2），我们始终只有一个 `CounterOp` 实例，因此也就只有一个 `count` 实例。

解决办法就是把 `count` 移到 `parent` 中：

~~~ java
public final class CounterOp<T 
implements Operator<Integer, T> {
    @Override
    public Subscriber call(Subscriber child) {
        Subscriber<T> parent = new Subscriber<T>() {
           int count;
    // ... the rest is the same
~~~

当然我们也有一些场景需要在订阅者之间共享变量，但这些场景少之又少，所以第一原则就是：`Operator` 的所有成员都声明为 `final`。一旦声明为 `final`，你很快就会发现你的代码在尝试修改它们（_你也很快就会发现代码写得有 bug_）。

## 8，Observable 链条中的共享状态（Shared state in an Observable chain）

假设你对 `toList()` 的性能不满意，或者它返回的 `List` 类型不满足需求，你打算实现一个自己的聚合器。你希望通过已有的操作符解决这个问题，你找到了 `reduce()`：

~~~ java
Observable<Vector<Integer>> list = Observable
    .range(1, 3)
    .reduce(new Vector<Integer>(), (vector, value) -> {
        vector.add(value);
        return vector;
    });
 
list.subscribe(System.out::println);
list.subscribe(System.out::println);
list.subscribe(System.out::println);
~~~

如果运行上面的代码，你会发现第一次打印符合预期，但第二次打印了两遍，第三次则打印了三遍！

问题不是出在 `reduce()` 本身，而是对它的使用方式。当链条建立起来之后，传入 `reduce()` 的 `Vector` 实例就相当于一个“全局”的了，后续对这个链条的调用都会共用同一个实例。

修复我们遇到的这个具体问题很简单，无需重新实现一个操作符：

~~~ java
Observable<Vector<Integer>> list2 = Observable
    .range(1, 3)
    .reduce((Vector<Integer>)null, (vector, value) -> {
        if (vector == null) {
            vector = new Vector<>();
        }
        vector.add(value);
        return vector;
    });
 
list2.subscribe(System.out::println);
list2.subscribe(System.out::println);
list2.subscribe(System.out::println);
~~~

你需要传入 `null`，并在聚合函数内创建新的 `Vector` 实例，这样就不会在订阅者之间共享了。

首要原则是，对于任何需要传入一个初始值的聚合操作符，都需要小心，很可能这个初始值是被不同订阅者所共享的，而如果你想要把链式调用的结果用多个订阅者去消费，它们就会发生冲突了，可能会导致不可预测的行为，甚至崩溃。

## 总结

在本文中，我分析了三种关于操作符更常见的陷阱，并且展示了如何测试并修复背后的 bug。

在使用 RxJava 各种操作符的过程中，可能很容易遇到各种奇怪（甚至滑稽）的问题（至少对我是这样），所以我的追求远未停止。而新的问题也变得越来越微妙，为了解决这些问题，我们需要了解更多关于操作符的原理。
