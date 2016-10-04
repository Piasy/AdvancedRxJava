---
layout: post
title: Subjects（二）：实现自定义 Subject 的要求、相关的工具类以及算法
tags:
    - Subject
---

原文 [Subjects (part 2)](http://akarnokd.blogspot.com/2015/09/subjects-part-2.html){:target="_blank"}

## 介绍

在本文中，我将讲讲编写一个 Subject 应该满足的要求，相关的工具类以及算法，最后我会根据上述内容实现一个支持 backpressure 的 Subject：`UnicastSubject`。

## 要求

由于 Subject 继承自 `Observable` 且实现了 `Observer`，所以它要遵循这两者的契约：

+ [Observer] `onXXX` 的调用需要是串行的。
+ [Observer] 应该遵循固定的模式：`onNext* (onError | onCompleted)?`。
+ [Observable] 订阅的调用应该是线程安全的。

此外，由于 Subject 会由于 `onError()` 和 `onCompleted()` 进入终结状态，我们也需要处理进入终结状态之后被 `Subscriber` 订阅的情况。让 Subscriber 傻等着显然不是个好主意。RxJava 的标准 Subject 都会向迟来的 Subscriber 重放终结事件（`ReplaySubject` 还会重放 `onNext`）。

考虑到上面的要求，我们的 `UnicastSubject` 只允许有一个 Subscriber 订阅，在被订阅之前，缓存先到达的事件，在被订阅之后，在保证 backpressure 的前提下，重放所有缓存的事件。

## UnicastSubject

先看看 `UnicastSubject` 的类结构：

~~~ java
public final class UnicastSubject<T> 
extends Subject<T, T> {
 
    public static <T> UnicastSubject<T> create() {   // (1)
        State<T> state = new State<>();
         
        return new UnicastSubject<>(state);
    }
     
    final State<T> state;                            // (2)
     
    protected UnicastSubject(State<T> state) {       // (3)
        super(state);
        this.state = state;
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
     
    @Override
    public boolean hasObservers() {
        // implement
    }
     
    static class State<T> implements
        OnSubscribe<T>, Observer<T>, 
        Producer, Subscription {                     // (4)
         
    }
}
~~~

对 RxJava 来说，Java 语言最大的不足就是没有扩展方法，扩展方法是指：一个方法看起来像是类 A 的成员函数，但其实它是类 B 的静态函数，但是编译器允许我们像是调用 A 的成员函数那样（fluent API）去调用它，编译器自动为我们调用类 B 的静态函数。所以在 Java 里面，要实现 fluent API，我们就需要一个包装类（`Observable`）来容纳所有的操作符以及函数。

由于当每个 Subscriber 订阅时，我们都需要进行不同的处理，所以 Observable 有一个 `protected` 构造函数，它接受一个 `OnSubscribe<T>` 回调（被订阅时使用这个回调处理 Subscriber）。然而，Subject 需要同时处理 `OnSubscribe<T>` 的调用以及 `onXXX` 调用。Java 不允许在调用父类构造函数之前调用非静态方法，所以下面的代码无法编译：

~~~ java
public class MyObservable<T> extends Observable<T> {
    protected MyObservable() {
        super(this::handleSubscriber);
    }
 
    void handleSubscriber(Subscriber<? super T> s) {
    }
}
~~~

解决办法有点别扭，但好歹可以解决问题：用一个共享的状态，它既作为 `OnSubscribe<T>`，也作为 `Observable` 的自身状态，然后用一个工厂方法把这种解决方案封装起来（1）。

_注意，RxJava 1.x 中大部分 Observable 的扩展，例如 Subject，都用了不同的对象来处理 Observable 状态和订阅处理，RxJava 2.x 改进了这一点，只用一个对象来处理这两个任务，正如我在这里展示的。_

_译者注：看到这里大家可能会觉得这违背了单一职责原则，是的，违背了，但收益却是性能的提升。在 RxJava 这样广泛使用的基础库中，性能至关重要，整个系列中，我们都看到了很多性能提升的细节，例如继承自 AtomicLong 类以节省一次内存分配（违背“组合优于继承”），不需要 volatile 时就不用，以节省几次 CPU 时钟周期。这些点我们知道就好，平时大部分的开发都用不着这样的优化，所以还是应该遵循 SOLID 等原则。_

有了 `State<T>` 对象之后，我们把它用作 `OnSubscribe<T>`，并且把它保存在 `UnicastSubject` 中（2，3）。`State` 实现了好几个接口（4）：

+ 实现 `OnSubscribe` 以处理每一次被订阅。
+ 实现 `Observer`，这样在 UnicastSubject 的 `onXXX` 方法中就可以转发到 State 中了。
+ 实现 `Producer`，由于我们知道只会处理一个 Subscriber，所以我们只需要一个 Producer，State 实现 Producer 就可以节省一次内存分配以及通信成本。
+ 实现 `Subscription`，以处理来自 child 的取消订阅，这里同样节省了一次内存分配以及通信成本。

`onXXX` 的实现很直观，全部转发给 State：

~~~ java
// ...
@Override
public void onNext(T t) {
    state.onNext(t);
}
 
@Override
public void onError(Throwable e) {
    state.onError(e);
}
 
@Override
public void onCompleted() {
    state.onCompleted();
}
 
@Override
public boolean hasObservers() {
    return state.child != null;
}
// ...
~~~

（_为了简洁起见，我就不在本文中实现 State 获取（state peeking）的逻辑了。_）

现在让我们编写 `State` 类，首先是添加一些成员变量：

~~~ java
static final class State<T> implements
    OnSubscribe<T>, Observer<T>, Producer, Subscription {
 
        volatile Subscriber<? super T> child;                  // (1)
         
        final AtomicBoolean once = new AtomicBoolean();        // (2)
         
        final Queue<T> queue = new SpscLinkedAtomicQueue<>();  // (3)
         
        volatile boolean done;                                 // (4)
        Throwable error;
         
        volatile boolean unsubscribed;                         // (5)
 
        final AtomicLong requested = new AtomicLong();         // (6)
 
        final AtomicInteger wip = new AtomicInteger();         // (7)
        // ...
~~~

成员还不少：

1. `child` 用来保存唯一的 Subscriber，当取消订阅之后，child 会被重置为 `null`。它必须是 `volatile` 的，因为 `hasObservers()` 需要线程安全地检查它是否为 `null`。
2. 我们需要确保在这个 Subject 的生命周期中，只有一个 Subscriber，我们利用一个 `AtomicBoolean` 实现。当然，我们也可以让 State 继承自 AtomicBoolean 以节省一次内存分配。还有另一种更复杂一点的方案，那就是直接利用 child，同时引入一个表示已被取消订阅状态的 Subscriber 常量。
3. `queue` 将会保存 Subscriber 到来之前收到的数据，或者表示 Subscriber 请求了一些数据。这里我用了一个单生产者单消费者的链表队列，但它的常量节点分配开销略大，在 RxJava 2.x 中我们进行了一些优化，使用了一个 `SpscLinkedArrayQueue`。
4. 我们保存了终结状态（包括可能的错误），由于错误只会在结束之前写一次，然后在结束之后读，所以它不需要是 `volatile`。
5. 由于 child 可能随时取消订阅，之后我们显然不必要继续保存新事件了。`unsubscribed` 和 `done` 将在 `onXXX` 中用来判断是否需要丢弃事件。
6. 我们需要记录 child 请求的数据量，保证不发出超量的数据（backpressure）。
7. `wip` 是在[前文提到的队列漏](/AdvancedRxJava/2016/05/13/operator-concurrency-primitives-2/){:target="_blank"}中要使用的，用来保证只有一个线程向 child 发送数据。

接下来，让我们实现 `OnSubscribe.call()` 并且实现对 Subscriber 的管理：

~~~ java
@Override
public void call(Subscriber<? super T> t) {
    if (!once.get() && once.compareAndSet(false, true)) {  // (1)
        t.add(this);
        t.setProducer(this);                               // (2)
        child = t;
        drain();                                           // (3)
    } else {
        if (done) {                                        // (4)
            Throwable e = error;
            if (e != null) {                               // (5)
                t.onError(e);
            } else {
                t.onCompleted();
            }
        } else {
            t.onError(new IllegalStateException(
                "Only one subscriber allowed."));          // (6)
        }
    }
}
~~~

1. 如果还没有 Subscriber 订阅过（`once` 为 `false`），且我们成功完成了 `once` 的 CAS 操作，那我们现在就已经迎来了唯一的 Subscriber。
2. 一定要先设置好取消订阅逻辑以及 Producer，再把 Subscriber 保存到 child，否则异步的 `onNext` 就可能在设置好之前先跑起来了。这在 RxJava 1.x 中没什么问题，但却违背了 reactive-streams 中 `Publisher` 的要求（_先设置好状态，再执行 `onXXX`_），现在保证这样便于我们以后进行移植。
3. 设置好状态之后，我们保存 child。不管此时 child 是否已经取消，我们都需要调用 `drain()` 函数，它会负责按需重放缓冲的数据以及清理操作。
4. 如果已经有了 Subscriber，那我们就检查自己（Subject）是否已经终止了。
5. 如果终止了，那我们就重放终止事件（onError/onCompleted），就像标准 Subject 那样。
6. 否则，我们就抛出 `IllegalStateException`。

接下来是 `onNext`，它可以实现得很简单，也可以很复杂，我们先看看简单的实现：

~~~ java
@Override
public void onNext(T t) {
    if (done || unsubscribed) {
        return;
    }
    queue.offer(t);
    drain();
}
~~~

如果我们没有终结或者被取消订阅，那我们就把数据加到队列中，然后调用 `drain()`（为了简洁起见，这里我再次省略了 `null` 检查）。

复杂的实现主要是加了一个快路径，在 child 已经就绪而且队列为空时，跳过队列的中转。但是注意，快路径并不会一直满足，因为 child 可以按照自己的需求进行请求，所以这时我们依然需要把数据加入到队列中。

~~~ java
@Override
public void onNext(T t) {
    if (done || unsubscribed) {
        return;
    }
    if (wip.get() == 0 && wip.compareAndSet(0, 1)) {  // (1)
        long r = requested.get();                     // (2)
        if (r != 0 && queue.isEmpty()) {              // (3)
            child.onNext(t);
            if (r != Long.MAX_VALUE) {                // (4)
                requested.decrementAndGet();
            }
            if (wip.decrementAndGet() == 0) {         // (5)
                return;
            }
        } else {
            queue.offer(t);                           // (6)
        }
    } else {
        queue.offer(t);                               // (7)
        if (wip.getAndIncrement() != 0) {
            return;
        }
    }
    drainLoop();                                      // (8)
}
~~~

这个实现看起来更有趣一些，它的工作原理如下：

1. 如果我们成功完成了对 wip 从 0 到 1 的转变，我们就进入了快路径。
2. 读取已请求的计数。
3. 我们需要检查是否有请求，而且队列为空。队列是否为空的检查至关重要，否则快路径就会跳过队列中的数据，进而打乱了数据的顺序。
4. 如果通过了检查，那我们就直接把数据发给 child。如果 child 不是处于无限请求模式，那我们就递减请求计数。
5. 然后我们递减 wip，如果为 0 了，那我们就可以退出了。如果不为 0，就说明有并发的 `request()`，那我们就需要继续处理它们。代码这时不会返回，而是继续（8）的漏循环。
6. 如果此时没有请求，或者队列不为空，那我们就把数据加入到队列中，然后进入（8）的漏循环。
7. 如果我们没有进入快路径，那我们就把数据加入到队列中，并且尝试增加 wip。如果递增之前 wip 不为 0，就说明有线程正在快路径中，它会最后进入漏循环，我们就可以返回了。而如果为 0，则说明没有线程在快路径中，那我们就需要进入漏循环。
8. 最后，我们在漏循环中进行队列漏操作。

`onError` 和 `onCompleted` 的实现和简单版的 `onNext` 差不多：

~~~ java
@Override
public void onError(Throwable e) {
    if (done || unsubscribed) {
        return;
    }
    error = e;
    done = true;
    drain();
}
 
@Override
public void onCompleted() {
    if (done || unsubscribed) {
        return;
    }
    done = true;
    drain();
}
~~~

非常直观，如果没有终结，就记录可能的异常，然后标记终结，最后 调用 `drain()`。

处理 child 的请求以及取消订阅也不复杂：

~~~ java
@Override
public void request(long n) {
    if (n < 0) {
        throw new IllegalArgumentException("n >= 0 required");
    }
    if (n > 0) {
        BackpressureUtils.getAndAddRequest(requested, n);      // (1)
        drain();
    }
}
 
@Override
public boolean isUnsubscribed() {
    return unsubscribed;
}
 
@Override
public void unsubscribe() {
    if (!unsubscribed) {
        unsubscribed = true;
        if (wip.getAndIncrement() == 0) {                      // (2)
            clear();
        }
    }
}
~~~

在 `request()` 中（1），我们增加请求数量到 `requested` 中，然后调用 `drain()`。`unsubscribe()` 稍微有趣一点（2）。我们设置 `unsubscribed` 标记（这里先检查后执行，并非原子操作，但是不会有问题，下面有对取消操作单线程的保证），然后增加 wip。增加 wip 有两点考虑，一是防止多个线程从这里开始执行清理操作，二是如果有线程在执行 `drainLoop()`，它会检查是否已经取消订阅，并帮我们进行清理操作。

继续，`clear()` 和 `drain()` 也很短：

~~~ java
void clear() {
    queue.clear();
    child = null;
}
 
void drain() {
    if (wip.getAndIncrement() == 0) {
        drainLoop();
    }
}
~~~

`clear()` 函数会清空队列，然后把 child 重置为 `null`。由于我们只会在 `unsubscribe()` 中 wip 的 CAS 成功之后才调用 `clear()`，所以 child 被重置之后就一定不会被使用了。`drain()` 函数只是递增 wip，而如果递增之前为 0，那就进入漏循环 `drainLoop()`。

好了，现在就是千呼万唤的 `drainLoop()` 了：

~~~ java
void drainLoop() {
    int missed = 1;                                           // (1)
     
    final Queue<T> q = queue;
    Subscriber<? super T> child = this.child;                 // (2)
     
    for (;;) {
         
        if (child != null) {                                  // (3)
             
            if (checkTerminated(done, q.isEmpty(), child)) {  // (4)
                return;
            }
             
            long r = requested.get();
            boolean unbounded = r == Long.MAX_VALUE;
            long e = 0L;                                      // (5)
             
            while (r != 0L) {
                boolean d = done;
                T v = q.poll();
                boolean empty = v == null;                    // (6)
                 
                if (checkTerminated(d, empty, child)) {
                    return;
                }
                 
                if (empty) {
                    break;
                }
                 
                child.onNext(v);
                 
                r--;
                e--;                                          // (7)
            }
             
            if (e != 0) {
                if (!unbounded) {
                    requested.addAndGet(e);                   // (8)
                }
            }
        }
         
        missed = wip.addAndGet(-missed);                      // (9)
        if (missed == 0) {
            return;
        }
         
        if (child == null) {                                  // (10)
            child = this.child;
        }
    }
}
~~~

再看队列漏的代码应该很熟悉了，让我们仔细分析一下：

1. 这里我们不是对 wip 逐次递减 1，而是批量减，首先我们假设我们只积累了一次 `drain()` 调用。后面如果我们积累了多次，`missed` 会更大，那我们就可以在（9）处一次性减完，能避免一些可能的空循环。
2. 我们把 `child` 和 `queue` 读到局部变量中，避免反复从成员变量中读。
3. 如果现在还没有 child Subscriber，那我们现在什么也不用做。
4. 我们需要检查是否已经到达终结状态。由于 `onError` 和 `onCompleted` 是直接发出不用管是否有请求量的，所以我们在读取请求量之前检查是否已经终结，如果是，那就退出循环。注意，要先检查 `done` 再检查队列是否为空，这一点很重要，因为是先把数据加入到队列中，再设置 `done` 的。
5. 读取请求量，检查是否是无限模式，以及准备好发射计数。
6. 我们先读取 done 的值，再从队列中取出一个数据，如果为 `null`，就说明队列已经空了。我们再次调用 `checkTerminated` 以确保及时响应取消订阅以及终结事件。
7. 我们递减请求计数和发射计数，通过使用递减，我们就可以在（8）处减少一次取负数操作。
8. 如果我们发出了数据，而且不是无限模式，我们就把请求计数减去发射数量 `e`（`e` 是负数，所以用 `addAndGet`）。
9. 所有当前的发射任务都完成之后，我们更新 wip。有可能此时 wip 依然不为 0，那我们就继续循环，否则就可以退出了。
10. 如果我们还要继续循环，就说明外面发生了点事情，那我们就重新读取 child，它是否为 `null` 这件事有可能不一样了。

好了，最后一个函数就是 `checkTerminated` 了。取决于是否需要延迟错误事件，这里有两种实现。如果错误需要延迟，那实现方式如下：

~~~ java
boolean checkTerminated(boolean done, boolean empty, 
        Subscriber<? super T> child) {
    if (unsubscribed) {                              // (1)
        clear();
        return true;
    }
    if (done && empty) {                             // (2)
        unsubscribed = true;
        this.child = null;
        Throwable e = error;
        if (e != null) {
            child.onError(e);
        } else {
            child.onCompleted();
        }
        return true;
    }
    return false;
}
~~~

首先我们检查 `unsubscribed`，如果已经取消订阅，那我们就清空队列，重置 child 为 null，并返回 true，表明漏循环应该退出（1）。否则，我们检查自己是否已经终结且队列为空，如果是，我们就标记 unsubscribed 为 true（方便后续处理），重置 child，并发出终结事件。否则我们就仍需继续漏循环（例如已终结，但队列中还有数据）。

另一种实现是尽早发出错误，忽略队列中的其他数据：

~~~ java
boolean checkTerminated(boolean done, boolean empty, 
        Subscriber<? super T> child) {
    if (unsubscribed) {
        clear();
        return true;
    }
    if (done) {
        Throwable e = error;                            // (1)
        if (e != null) {
            unsubscribed = true;
            clear();
            child.onError(e);
            return true;
        } else if (empty) {                             // (2)
            unsubscribed = true;
            this.child = null;
            child.onCompleted();
            return true;
        }
    }
    return false;
}
~~~

如果已经终结，我们就检查是否有错误。如果有错误，那就清空队列，并通过 `clear()` 重置 child，最后发出错误（1）。否则检查队列是否为空，因为我们应该先发完数据（2）。由于我们已经知道队列空了，所以就不用调用 `clear()` 了。

## 总结

在本文中，我实现了一个 `UnicastSubject`，它符合 backpressure 的要求，并且只允许有一个 Subscriber 订阅。

在下一篇文章中，我将重新实现 `PublishSubject` 以支持多个 Subscriber。
