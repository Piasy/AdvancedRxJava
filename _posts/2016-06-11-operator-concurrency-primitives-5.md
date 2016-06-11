---
layout: post
title: Operator 并发原语： producers（三）
tags:
    - Operator
    - Producer
---

原文 [Operator concurrency primitives: producers (part 3)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_9.html){:target="_blank"}

## 介绍

在本文中，我将实现一种 producer，它可以按需发出**多个数据**，同时保证下游的 backpressure 功能正常工作。

我称之为 **queued-producer**，因为它有一个待发数据的队列，并且在下游请求时重放其中的部分数据。

这种 producer 可能的使用场景有以下几种：

1. 它能用来隔离发射速度快的 producer 和消费速度慢的 consumer，例如：`onBackpressureBuffer()`。
2. 它可以把最后的 n 个数据放入队列，并且在上游调用 `onCompleted()` 之后发出数据，例如：`takeLast()`。
3. 当你希望在上游发出一个数据之后，发出多个数据时，例如：某种类型的 `merge()` 操作。

## 只支持数据事件的 queued-producer

queued-producer 支持终止事件比较复杂，所以最开始我先实现一个更为简单的版本，它只需要处理数据事件。你可以用 `Notification` 或者 `NotificationLite` 来包装 `onError` 和 `onCompleted` 事件，当然除非你使用性能更差的 `ConcurrentLinkedDeque`，否则你是不可能支持 `onError` 的。

一如寻常，我先继承自 `AtomicLong` 来保存下游的并发请求的数量：

~~~ java
public final class QueuedProducer<T> 
extends AtomicLong implements Producer {
    private static final long serialVersionUID = -1;
     
    final Subscriber<? super T> child;
    final Queue<T> queue;
    final AtomicInteger wip;
     
    public QueuedProducer(Subscriber<? super T> child) {
        this.child = child;
        this.queue = new SpscLinkedQueue<>();
        this.wip = new AtomicInteger();
    }
     
    @Override
    public void request(long n) {                              // (1)
        // handle request
    }
     
    public void offer(T value) {                               // (2)
        // handle a value offered
    }
     
    private void drain() {                                     // (3)
        // attempt to drain
    }
}
 
Observable<Integer> source = Observable.create(child -> {
    QueuedProducer<Integer> qp = new QueuedProducer<>(child);
    child.setProducer(qp);
    for (int i = 0; i < 200; i++) {
        qp.offer(i);
    }
});
 
source.take(150).subscribe(System.out::println);
~~~

`QueuedProducer` 类的结构很直观，而且和之前文章中的 producer 结构很像。我们在（1）处理下游的请求，在（2）处理上游发过来的数据，新鲜的只有（3）处的 `drain()` 函数。正如它的名字一样，它是用来实现[前文中](/AdvancedRxJava/2016/05/13/operator-concurrency-primitives-2/){:target="_blank"}所介绍的队列漏（queue-drain）的。为了简洁起见，这里我将省略快路径（fast-path）的优化。

首先让我们实现 `request()` 方法：

~~~ java
// ...
@Override
public void request(long n) {
    if (n < 0) {
        throw new IllegalArgumentException();   // (1)
    }
    if (n > 0) {
        BackpressureUtils
            .getAndAddRequest(this, n);         // (2)
        drain();                                // (3)
    }
}
// ...
~~~

它看起来并不复杂，但这也意味着肯定还有其他的地方很复杂。经过（1）处的常规检查之后，我们通过原子操作增加了 `AtomicLong` 的值（2），增加成功之后调用 `drain()` 方法。

幸运的是，`offer()` 方法比 `request()` 还要简单：

~~~ java
// ...
public void offer(T value) {
    queue.offer(Objects.requireNonNull(value));  // (1)
    drain();                                     // (2)
}
// ...
~~~

我们首先在（1）往队列中加入不为 null 的数据（[JCTools](https://github.com/JCTools/JCTools){:target="_blank"} 不允许插入 `null`），然后我们调用 `drain()` 函数（2）。注意，我们使用的 queue 类型是 `SpscLinkedQueue`，这意味着我们假设 `offer()` 方法将会被串行调用，而这也正是 `QueuedProducer` 的使用场景。但是 `QueuedProducer` 的逻辑在多个线程并发调用 `offer()` 时也是正确的，此时我们只需要使用 `MpscLinkedQueue` 即可。

最后，我们就只剩下 `drain()` 函数了：

~~~ java
    // ...
    private void drain() {
        if (wip.getAndIncrement() == 0) {                // (1)
            do {
                if (child.isUnsubscribed()) {            // (2)
                    return;
                }
 
                wip.lazySet(1);                          // (3)
                 
                long r = get();                          // (4)
                long e = 0;
                T v;
                 
                while (r != 0 &&                         // (5)
                        (v = queue.poll()) != null) {    // (6)
                    child.onNext(v);
                    if (child.isUnsubscribed()) {        // (7)
                        return;
                    }
                    r--;
                    e++;
                }
                 
                if (e != 0) {                            // (8)
                    addAndGet(-e);
                }
            } while (wip.decrementAndGet() != 0);        // (9)
        }
    }
}
~~~

代码结构看起来应该很熟悉，因为它基本上就是之前文章中描述的队列漏，只不过它处理了更多的状态：

1. 如果我们成功执行了 `wip` 从 0 到 1 的自增操作，我们就进入漏循环。而如果自增前不为 0，则相当于告诉正在执行漏循环的线程，还有更多请求需要处理。
2. 在循环中，我们首先检查 child 是否已经取消订阅。
3. 我们把 `wip` 的值重置为 1，这意味着我们开始解决待处理的请求了。通过这样做，我们通常可以省下一些外层循环的执行，直接跳过那些没有数据可以发射的情况（_译者注：因为我们在内循环中会尝试把所有的请求都处理完毕，所以即便 `wip` 不为 0，我们也没有数据可以发射，所以直接跳过这些情况可以节约时间_）。使用 `lazySet` 方法，我们可以省去一些 CPU 周期，因为后续的操作符相当于一个完整的栅栏（barrier）（_它会保证逻辑的正确_），而我们只关心 `wip` 是否非零，并不关心其具体的值。
4. 我们读取当前被请求的数量，因为我们只应发射这么多数据。
5. 在内循环中，我们发射尽可能多的数据，当然，我们受制于下游请求的数量，以及上游给我们的数据量。
6. 当然我们也可以用 `!queue.isEmpty()` 来进行判断，但是 `poll()` 能够让我们在一步内完成判断是否为空以及获取下一个数据的操作（得益于 queue 不允许 `null`）。
7. 发射数据之后，我们检查 child 是否已经取消订阅。
8. 我们在栈内统计当前已经发射的数量，并一次性从总请求数量中减去（而不是在内循环中每次减 1）。
9. 最后，我们递减 `wip`，如果递减之后为 0，我们就可以安全退出循环了。

_注意：上述的发射循环还存在优化空间，例如，增加一个快路径，直接跳过队列的中转，或者 `request()` 函数无需调用 `drain()` 函数，只需要增加 n 即可。目前我并未探究这些优化的并发特性。_

## 完整的 queued-producer

既然我们已经实现了只支持数据事件的 queued-producer，现在我们就可以扩展它使之可以处理 `onError` 和 `onCompleted` 了，而且还能让 `onError` 跳过请求的数据，直接终止 producer 的执行。

`FullQueuedProducer` 将实现 `Producer` 和 `Observer` 接口，并且继承自 `AtomicLong`：

~~~ java
public final class FullQueuedProducer<T> 
extends AtomicLong implements Producer, Observer<T> {
    private static final long serialVersionUID = -1L;
     
    final Subscriber child;
    final Queue<T> queue;
    final AtomicInteger wip;
     
    Throwable error;                                         // (1)
    volatile boolean done;                                   // (2)
     
    public FullQueuedProducer(Subscriber child) {
        this.child = child;
        this.queue = new SpscLinkedQueue<>();
        this.wip = new AtomicInteger();
    }
     
    @Override
    public void request(long n) {
        if (n < 0) {
            throw new IllegalArgumentException();
        }
        if (n > 0) {
            BackpressureUtils.getAndAddRequest(this, n);
            drain();
        }
    }
     
    @Override
    public void onNext(T value) {                             // (3)
        queue.offer(Objects.requireNonNull(value));
        drain();
    }
     
    @Override
    public void onError(Throwable e) {                        // (4)
        error = e;
        done = true;
        drain();
    }
     
    @Override
    public void onCompleted() {                               // (5)
        done = true;
        drain();
    }
     
    private boolean checkTerminated(boolean isDone, 
            boolean isEmpty) {
        // implement
    }
 
    private void drain() {
        // different implementation
    }
}
~~~

类的基本结构有些不同了，而且我们需要一个不同的 `drian()` 实现：

1. 我们保存一个 `Throwable` 类型的成员 `error`，任何不为 `null` 的值都表明了异常退出，而当 `done` 置为 `true` 且 `error` 为 `null` 时则意味着正常退出。
2. `done` 置为 `true` 则意味着上游不会有任何新的数据或者其他事件了。这依赖于 `FullQueuedProducer` 的 `onXXX` 方法会被串行调用的假设。注意 `done` 必须声明为 `volatile`，因为在 `drain()` 函数中我们需要检查是否要提前结束，而 `wip` 无法保证我们每次都能读取到最新的值。
3. 我们将 `QueuedProducer` 中的 `offer()` 函数改成了 `onNext`。
4. 如果发生了错误，我们保存异常对象的引用，把 `done` 置为 `true`，并调用 `drain()` 函数。
5. 如果上游正常结束，我们就把 `done` 置为 `true`，并调用 `drain()` 函数。

新的 `drain()` 方法有了更多需要检查和操作的东西，首先，我将实现一个辅助函数，来检查 `FullQueuedProducer` 是否需要结束执行：

~~~ java
// ...
private boolean checkTerminated(
        boolean isDone, boolean isEmpty) {
    if (child.isUnsubscribed()) {                  // (1)
        return true;
    }
    if (isDone) {                                  // (2)
        Throwable e = error;                       
        if (e != null) {                           // (3)
            queue.clear();
            child.onError(e);
            return true;
        } else if (isEmpty) {                      // (4)
            child.onCompleted();
            return true;
        }
    }
    return false;                                  // (5)
}
// ...
~~~

1. 首先，如果 child 已经取消订阅，我们就直接返回 `true`，这时我们会退出执行。
2. 然后，如果 `isDone` 为 `true`，我们需要检查当前处于何种状况，并执行相应的退出逻辑。之所以通过一个 `isDone` 参数而不是直接读取 `done` 的值，是因为判断队列是否为空必须在检查 `done` 之后，这一点我稍后将详细解释。
3. 由于我们假设上游的调用是串行的，所以我们可以直接读取 `error` 的值，这是因为对 `error` 的赋值在对 `volatile done` 的赋值之前，而对 `error` 的读取在对 `done` 的读取之后，所以我们形成了一种对 `done` 的恰当 acquire-release 操作（_啥意思？不太懂..._）。如果 `error` 不为 `null`，我们就会把它传递给 child，并且返回 `true`。同时队列也会被清空，以免其中的数据被继续引用。
4. 如果此时队列为空，我们就调用 child 的 `onCompleted` 函数，并且返回 `true`。
5. 其他情况下都返回 `false`，表示循环应该继续执行。

最后就是 `FullQueuedProducer` 的 `drain()` 函数了：

~~~ java
    // ...
    private void drain() {
        if (wip.getAndIncrement() == 0) {
            do {
                if (checkTerminated(done, queue.isEmpty())) {    // (1)
                    return;
                }
                 
                wip.lazySet(1);
                 
                long r = get();
                long e = 0;
 
                while (r != 0) {
                    boolean d = done;                            // (2)
                    T v = queue.poll();
                    if (checkTerminated(d, v == null)) {         // (3)
                        return;
                    } else if (v == null) {                      // (4)
                        break;
                    }
                     
                    child.onNext(v);
                    r--;
                    e++;
                }
                 
                if (e != 0) {
                    addAndGet(-e);
                }
            } while (wip.decrementAndGet() != 0);
        }
    }
}
~~~

乍一看，和 `QueuedProducer` 的实现很相似，但实际上却存在重要的差异：

1. 在执行漏循环之前，我们不仅仅是检查 `isUnsubscribed`，而是检查是否存在异常，或者上游已经结束。注意，检查 `queue` 是否为空发生在读取 `done` 的值之后。
2. 在检查 `queue` 中的数据（同时检查是否为 `null`）之前，我们先读取 `done` 的值。
3. 然后我们用 `d` 的值和 `v` 是否为 `null` 来调用 `checkTerminated`。
4. 即便我们无需退出执行，队列也是有可能为空的，此时我们就需要退出内循环了。

为什么我们需要在检查队列是否为空之前读取 `done` 的值？因为 `drain()` 函数可能被并发调用（被 `onXXX()` 函数或者 `request()` 函数调用），而如果不保证这一点，`drain()` 函数在检查队列是否为空和检查 `done` 之间的暂停，将给并发的上游一个时间窗口，让上游可以传递过来一些数据，然后调用 `onCompleted`。一旦 `drain()` 函数再次恢复执行，此时看到的 `done` 是 `true` 了，我们就会结束执行，这时就发生了事件丢失了。（_译者注：而如果我们在检查队列是否为空之前先读取 `done` 的值，那上述情况下，我们看到的值就仍然是 `false`，我们就仍会把队列中的数据都发射给下游，不存在事件丢失的情况_）

## 总结

实现一个支持 backpressure 且能发射多个数据的 producer，最初看起来可能很复杂，但如果你阅读过之前的文章，而且熟悉 Java 基本的并发编程技巧，以及一些内存模型的基本概念，实现起来也没有想象中那么复杂，对不对？我认为理解这一 producer 的内部工作原理非常重要，因为 RxJava 中高级的操作符都是基于同样的队列漏逻辑实现的，并且和 `FullQueuedProducer` 有着同样的结束状态管理逻辑，例如：`publish()`。

在下一篇文章中，我将再次讲解 `RangeProducer`，关于如何为其发射循环增加一个**快路径（fast-path）**，用于当下游没有实现 backpressure，并且向上游请求 `Long.MAX_VALUE` 时，避免不必要的状态管理。
