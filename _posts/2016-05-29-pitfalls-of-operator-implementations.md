---
layout: post
title: 实现操作符时的一些陷阱（一）
tags:
    - Operator
---

原文 [Pitfalls of operator implementations (part 1)](http://akarnokd.blogspot.com/2015/05/pitfalls-of-operator-implementations.html){:target="_blank"}

## 介绍
早些时候，实现一个操作符（`Operator`）是一个相对简单的事情：只需要为上游准备一个被操控的订阅者，并且在 `onXXX` 方法中实现需要的逻辑即可。

然而大家很快意识到，这种做法有的时候行不通，例如上游是同步且无限事件流时，一旦用户再使用 `take(n)`，就会出问题了。解决办法就是把一个取消订阅的令牌（unsubscription token）注入到事件流中，而且上游的 'producers' 需要检查 `isUnsubscribed()` 的结果并且据此停止自己的生产行为。这一点有点奇怪，因为最初由 Erik Meijer 在 Channel 9 的视频中描述的迭代器的二元化（Iterator-dualization）中，并没有预料到这一问题的存在（正如他所言，“麻烦事儿才刚开始（that's where the heavy handwaving starts）”）。

第二个大问题（_译者注：原文作者称之为“复杂性休克”（complexity shock），我在此处就简化一下了_）在支持 backpressure 以及引入 `Producer` 接口时出现了。如果有人觉得取消订阅很复杂，那 backpressure 就是相当困难了。它困难到即使是我（依据 git blame，我为 RxJava 贡献了 27% 的代码）也不能保证 100% 正确。

由于绝大部分的操作符都需要处理这两个问题，所以我决定暂停对 producer 的介绍，先介绍一下操作符编写者经常陷入的几个最常见的陷阱，他们常常在提交 pull request 时或者在 StackOverflow 问问题时，落入这些陷阱。

## 1，打破了取消订阅和 backpressure 的链条（Breaking the chain of unsubscription and backpressure）

像 `map()` 这样的操作符，它们会把原事件一一映射为新的事件，但它们本身不会直接影响取消订阅和 backpressure。

例如，如果有人想要编写一个“优化的”（"optimized"）操作符，把一个整数根据其奇偶性映射为一个 boolean，他可能会这样实现：

~~~ java
Operator<Boolean, Integer> isOdd = child -> {
    return new Subscriber<Integer>() {                    // (1)
        @Override
        public void onNext(Integer value) {
            child.onNext((value & 1) != 0);
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
};
 
Observable.range(1, 2_000_000_000)
    .lift(isOdd)
    .take(2)
    .subscribe(System.out::println);
~~~

如果我们运行上述程序，我们会发现，在打印了 `true` 和 `false` 之后，程序继续运行了一段时间。`take()` 操作符取消订阅的请求没有到达 `range()` 操作符。在这个例子中，这个问题只是让程序多运行了几秒钟，但其他情况下，`lift` 前面的操作符可能非常消耗计算资源，因此会占用远比正常情况下要多的 CPU 和其他资源。

问题出在（1）处：`child` 已经被取消订阅了，但我们没有通知上游终止运行。我们可以利用 `Subscriber(Subscriber<?> op)` 函数来解决这个问题，它会为我们的 subscriber 和 child subscriber 建立联系：

~~~ java
Operator<Boolean, Integer> isOdd = child -> {
    return new Subscriber<Integer>(child) {
    // the rest is the same
~~~

## 2，取消订阅了下游（Unsubscribing the downstream）

像 `take()` 这样的操作符，可能会在上游发出 `onCompleted()` 之前就终止了事件流。

例如，有人可能会实现一个 `takeNone()` 操作符，它在第一个 `onNext` 事件到来时发出 `onCompleted` 事件，同时取消订阅自己，以终止上游的执行。当然，我们吸取了上一条的教训，我们会把订阅者牢牢联系在一起：

~~~ java
Operator<Integer, Integer> takeNone = child -> {
    return new Subscriber<Integer>(child) {          // (1)
        @Override
        public void onNext(Integer t) {
            child.onCompleted();
            unsubscribe();                           // (2)
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
};
 
TestSubscriber<Integer> ts = new TestSubscriber<>();
 
Subscription importantResource = Subscriptions.empty();
ts.add(importantResource);
 
Observable.range(1, 100).lift(takeNone).unsafeSubscribe(ts);
 
if (importantResource.isUnsubscribed()) {
    System.err.println("Somebody unsubscribed our resource!");
}
~~~

在上述例子中，尽管我们使用了 `unsafeSubscribe()` 来阻止 `ts` 的自动取消订阅，`importantResource` 还是被取消订阅了。问题就出在“过度联系”（'over-chaining'）。由于（1）处的 `Subscriber(Subscriber<?> op)`，我们已经和 `importantResource` 复用了同一个复合 subscription（composite subscription），所以尽管看起来在（2）处我们只取消订阅了我们自己的 `Subscriber`，我们也会一起把 child 也取消订阅了。有人可能认为这个例子太过刻意，但实际上这个问题影响了 RxJava 的很多操作符，尤其是那些会在 `onCompleted()` 中执行特定逻辑的操作符，例如 `toList()`，`observeOn()`。

要解决这个问题，我们可以使用另一个 subscriber 的构造函数：`Subscriber(Subscriber<?> op, boolean shareSubscriptions)`。

~~~ java
Operator<Integer, Integer> takeNone = child -> {
    Subscriber<Integer> parent = 
            new Subscriber<Integer>(child, false) {   // (1)
        @Override
        public void onNext(Integer t) {
            child.onCompleted();
            unsubscribe();
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
    child.add(parent);                             // (2)
    return parent;
};
~~~

在这种方案中，首先我们在（1）处打破了取消订阅的链条，但是保持了 backpressure 的链条，然后我们把新的 subscriber 加入到 child 中。这会重新建立取消订阅的链条，如果 child 取消订阅了，我们也将会被取消订阅，进而取消订阅上游，但如果我们在 `onNext()` 中调用 `unsubscribe()`，我们只会取消自己以及上游，下游并不会被取消订阅。

_注：我曾力争把上述的这种模式作为所有操作符的默认模式，除非被证明不必要，但是我的意见和例子并未被 RxJava 通过，很多操作符维持了原有的模式，就像 `takeNone`。你可以猜一下有多少关于缺失的事件或者非正常取消订阅的 issue 最终确认是由上面的问题导致的。_

## 3，忘记请求更多（Forgetting to request more）

很多涉及 backpressure 的操作符，都期待请求的数据会在请求更多（request more）之前到达。例如要编写一个过滤掉所有奇数的操作符，有人可能会这样实现：

~~~ java
Operator<Integer, Integer> evenFilter = child -> {
    return new Subscriber<Integer>(child) {
        @Override
        public void onNext(Integer t) {
            if ((t & 1) == 0) {
                child.onNext(t);
            }
                                                        // (1)
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
};

Observable.range(1, 2).lift(evenFilter).subscribe(
        System.out::println,
        Throwable::printStackTrace,
        () -> System.out.println("Done"));
        
Observable.range(1, 2).lift(evenFilter).take(1).subscribe(
        System.out::println,
        Throwable::printStackTrace,
        () -> System.out.println("Done"));
~~~

第一个订阅（observation）会打印 `2` 和 `Done`，符合我们的预期，但第二个订阅却没有打印任何内容，尽管它们在功能上应该是一样的，因为 `range(1, 2)` 中只存在一个偶数。问题出在（1）处：当我们的操作符丢弃了一个数据之后，它并没有请求另一个数据。因此上游无法知道我们是否丢弃了一个数据，因此我们把 `take(1)` 直接发给了上游，这时上游在没有收到新的请求时，不会执行任何操作。所以解决办法是在我们丢弃一个数据的同时，请求一个新的数据：

~~~ java
// ... same as before
@Override
public void onNext(Integer t) {
    if ((t & 1) == 0) {
        child.onNext(t);
    } else {
        request(1);
    }
}
// ... same as before
~~~

修改之后，两个订阅都会打印 `2` 和 `Done`，符合我们的预期。

## 4，多次结束（Completing again）

有的操作符对两个事件流执行操作，但是第二个流决定结果事件流的状态，例如 `takeUntil()`。例如有人可能会实现一个操作符，它会延迟一个事件流的事件，直到另一个发出了一个零或者结束事件：

~~~ java
Observable<Integer> other = Observable.<Integer>create(o -> {
    try {
        Thread.sleep(100);
    } catch (Throwable e) {
        o.onError(e);
        return;
    }
    o.onNext(0);
    o.onCompleted();
}).subscribeOn(Schedulers.io());

Operator<Integer> takeUntilZero = child -> {
    Subscriber<Integer> main = 
            new Subscriber<Integer>(child, false) {
        @Override
        public void onNext(Integer t) {
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
    };
    Subscriber<Integer> secondary = new Subscriber<Integer>() {
        @Override
        public void onNext(Integer t) {
            if (t == 0) {
                onCompleted();
            }
        }
        @Override
        public void onError(Throwable e) {
            child.onError(e);
        }
        @Override
        public void onCompleted() {                 // (1)
            child.onCompleted();
            main.unsubscribe();
            unsubscribe();
        }
    };
    child.add(main);
    child.add(secondary);
    
    other.unsafeSubscribe(secondary);
    
    return main;
};

Observable<Integer> source = 
        Observable.timer(30, 30, TimeUnit.MILLISECONDS)
        .map(v -> v.intValue());

source.lift(takeUntilZero).unsafeSubscribe(
        Subscribers.create(
            System.out::println,
            Throwable::printStackTrace,
            () -> System.out.println("Done")
        )
);

Thread.sleep(1000);
~~~

如果运行这个例子，你会发现 `Done` 被打印了两次。问题就出在（1）处，我们为了实现另一个流发出了一个零或者结束事件时结束最终的事件流。由于 observable 忽略取消订阅的请求并继续发出事件是符合规范的，所以 `other` 可以在发出 `0` 之后直接调用 `onCompleted()`，而这就会导致 `secondary` 执行两次 `onCompleted()`（_一次是收到 `0` 之后自己调用，另一次是 `other` 调用的 `onCompleted()`_），并最终传递到了 `child`。默认情况下，RxJava 有一些保护措施，可以让最终的用户免受这样的错误影响，但是我们想要编写高性能的操作符，所以我们抛弃了这些保护措施，所以我们在这里使用了 `unsafeSubscribe()`。解决方法就是引入一个 boolean 值，记为 `done`，如果执行过 `onCompleted()` 了，我们就不再执行第二次：

~~~ java
// ... same as before
Subscriber<Integer> secondary = new Subscriber<Integer>() {
    boolean done;
    @Override
    public void onNext(Integer t) {
        if (t == 0) {
            onCompleted();
        }
    }
    @Override
    public void onError(Throwable e) {
        child.onError(e);
    }
    @Override
    public void onCompleted() {
        if (!done) {
            done = true;
            child.onCompleted();
            main.unsubscribe();
            unsubscribe();
        }
    }
};
// ... same as before
~~~

## 5，忘记了串行访问（Forgetting to serialize）
在上面**多次结束**的例子中，还有一个隐藏很深的 bug，简单运行几次难以发现，对 `child` 的 `onXXX` 方法的调用，可能发生在任何线程任何时间，而这违反了 RxJava 的约定：`Observer` 的方法的调用必须是串行的。

为了阻止这种问题的发生，我们需要进行一定的串行访问控制，但使用[前面文章中提到的串行访问方式](/AdvancedRxJava/2016/05/06/operator-concurrency-primitives/){:target="_blank"}就有点矫枉过正了，RxJava 有一个 `SerializedSubscriber` 专门用于这种需求：

~~~ java
// ... same as before
Operator<Integer, Integer> takeUntilZero = c -> {
    Subscriber<Integer> child = new SerializedSubscriber<>(c);
// ... same as before
~~~

仅仅是把 lambda 表达式的参数重命名为 `c`，并用 `SerializedSubscriber` 包装一下，我们就满足了串行访问的要求。

## 总结

在本文中，我展示了实现操作符时最常见最基本的陷阱，并且展示了如何发现问题，并且修复问题。确实，我们都希望每个操作符都有一小段可执行的代码，就相当于操作符的单元测试，尤其是有人想要引入新的操作符时。

还有一些其他的陷阱，尤其是涉及到 backpressure 时，但它们通常都是由对 backpressure 和 `Producer` 应该如何在复杂操作符中协调工作理解不深刻导致的。

经过这段小插曲之后，我将会继续介绍几种典型的高级 Producer：**single-producers** 和 **single-delayed-producers**。
