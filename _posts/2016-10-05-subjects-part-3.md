---
layout: post
title: Subjects（三，完结）：支持 backpressure 的 PublishSubject
tags:
    - Subject
---

原文 [Subjects (part 3 - final)](http://akarnokd.blogspot.com/2015/09/subjects-part-3-final.html){:target="_blank"}

## 介绍

本文作为 Subject 系列最后一篇文章，将会实现一个 `PublishSubject` 的变体。为了让这个过程更有趣，这个 `PublishSubject` 会遵循 backpressure，当订阅者请求得没那么快时，不会将它“淹没”。

## PublishSubject

`PublishSubject` 的主要类结构和上一篇文章中的 `UnicastSubject` 非常相似，所以我就直接跳过，重点关注在 `State` 类上。

`State` 的变化就会非常大了，因为 PublishSubject 允许同时存在多个 Subscriber，每个 Subscriber 都会有自己的 `unsubscribed`，`requested` 和 `wip` 状态。所以 State 不能直接实现 `Producer` 和 `Subscription` 了。我们会用另一个类实现它们：`SubscriberState`，每个 Subscriber 都会有一个 SubscriberState。

在讲 SubscriberState 的细节之前，还需要提一下 backpressure 处理策略。我们要支持用户指定 backpressure 策略，这样他们就不用再手动使用 `onBackpressureXXX` 了。为此，我们定义了 3 个 `enum` 值：

~~~ java
enum BackpressureStrategy {
    DROP,
    BUFFER,
    ERROR
}

public static <T> PublishSubject<T> createWith(
        BackpressureStrategy strategy) {
    State<T> state = new State<>(strategy);
    return new PublishSubject<>(state);
}
~~~

它们的名字就说明了功能：丢弃过多的数据，缓冲过多的数据，或者向 Subscriber 发出错误事件。

现在让我们看看 `State` 类的结构：

~~~ java
static final class State<T> 
implements OnSubscribe<T>, Observer<T> {
    final BackpressureStrategy strategy;
     
    @SuppressWarnings("unchecked")
    volatile SubscriberState<T>[] subscribers = EMPTY;
     
    @SuppressWarnings("rawtypes")
    static final SubscriberState[] EMPTY = new SubscriberState[0];
 
    @SuppressWarnings("rawtypes")
    static final SubscriberState[] TERMINATED = 
        new SubscriberState[0];
     
    volatile boolean done;
    Throwable error;
     
    public State(BackpressureStrategy strategy) {
        this.strategy = strategy;
    }
     
    boolean add(SubscriberState<T> subscriber) {
        // TODO Auto-generated method stub
    }
     
    void remove(SubscriberState<T> subscriber) {
        // TODO Auto-generated method stub
    }
     
    Subscriber<T>[] terminate() {
        // TODO Auto-generated method stub    
    }
     
     
    @Override
    public void call(Subscriber<? super T> t) {
        // TODO Auto-generated method stub
    }
     
    @Override
    public void onNext(T t) {
        // TODO Auto-generated method stub
    }
     
    @Override
    public void onError(Throwable e) {
        // TODO Auto-generated method stub
    }
     
    @Override
    public void onCompleted() {
        // TODO Auto-generated method stub
    }
}
~~~

现在还没有特殊的地方。我们用一个 `volatile SubscriberState` 数组保存所有订阅者的状态，`add`，`remove` 和 `terminate` 方法进行操作。我们利用 `EMPTY` 常量，避免每次所有的 Subscriber 都取消之后都要分配一个新的空数组。这种方式看过[此前 Subscription 容器相关文章](/AdvancedRxJava/2016/07/29/operator-concurrency-primitives-subscription-containers-3/index.html){:target="_blank"}的朋友应该会很熟悉。现在让我们看看 `add()` 的实现：

~~~ java
boolean add(SubscriberState<T> subscriber) {
    synchronized (this) {
        SubscriberState<T>[] a = subscribers;
        if (a == TERMINATED) {
            return false;
        }
        int n = a.length;
         
        @SuppressWarnings("unchecked")
        SubscriberState<T>[] b = new SubscriberState[n + 1];
         
        System.arraycopy(a, 0, b, 0, n);
        b[n] = subscriber;
        subscribers = b;
        return true;
    }
}
~~~

为了让我展现的实现方式多样化，这里我用了 `synchronized` 来进行同步，并没有使用 CAS 循环。上面的代码就是一个 copy-on-write 操作。这一实现方式的优点就是对数组进行遍历的时候会更快，而且基于一个经验事实，大部分 Subject 都不会同时有多个 Subscriber。但是如果我们真的遇见了会有大量 Subscriber 的场景，我们可以在同步代码块内使用基于 List 或者 Set 的容器。这里也有一个缺点，就是我们需要线程安全地对集合进行遍历，而这里唯一的线程安全方式就是进行一次深拷贝。

让我们接着看 `remove()` 的实现：

~~~ java
@SuppressWarnings("unchecked")
void remove(SubscriberState<T> subscriber) {
    synchronized (this) {
        SubscriberState<T>[] a = subscribers;
        if (a == TERMINATED || a == EMPTY) {
            return;
        }
        int n = a.length;
         
        int j = -1;
        for (int i = 0; i < n; i++) {
            if (a[i] == subscriber) {
                j = i;
                break;
            }
        }
         
        if (j < 0) {
            return;
        }
        SubscriberState<T>[] b;
        if (n == 1) {
            b = EMPTY;
        } else {
            b = new SubscriberState[n - 1];
            System.arraycopy(a, 0, b, 0, j);
            System.arraycopy(a, j + 1, b, j, n - j - 1);
        }
        subscribers = b;
    }
}
~~~

同样也是 copy-on-write 的实现方式，也利用了 EMPTY 常量。

接下来我们看看 `terminate()`：

~~~ java
@SuppressWarnings("unchecked")
SubscriberState<T>[] terminate() {
    synchronized (this) {
        SubscriberState<T>[] a = subscribers;
        if (a != TERMINATED) {
            subscribers = TERMINATED;
        }
        return a;
    }
}
~~~

这里我们检查当前是否处于终结状态，如果不是，就把 subscribers 置为 TERMINATED，并且返回之前的值。

现在我们就可以实现 `call()` 了：

~~~ java
@Override
public void call(Subscriber<? super T> child) {
    SubscriberState<T> innerState = 
        new SubscriberState<>(child, this);            // (1)
    child.add(innerState);                             // (2)
    child.setProducer(innerState);
     
    if (add(innerState)) {                             // (3)
        if (strategy == BackpressureStrategy.BUFFER) { // (4)
            innerState.drain();
        } else if (innerState.unsubscribed) {          // (5)
            remove(innerState);
        }
    } else {
        Throwable e = error;                           // (6)
        if (e != null) {
            child.onError(e);
        } else {
            child.onCompleted();
        }
    }
}
~~~

1. 我们创建一个 `SubscriberState` 把 child 包装起来，这样对每个 Subscriber 的事件分发处理就是独立的。
2. 我们把 SubscriberState 加入到 child 中，用于取消订阅和请求处理。
3. 我们把 `innerState` 加入到 `subscribers` 数组中，当然这一步可能失败，这就说明 Subject 自身已经被并发的终结了。
4. 如果我们当前的 backpressure 策略是 BUFFER，那我们就要启动漏循环了。
5. 即便 `add()` 成功，child 也有可能被并发的取消订阅了，这时我们就需要尝试把它移除掉。
6. 如果（3）处的 `add()` 失败，就说明此时 Subject 已经终结了，那我们就需要向 child 发送终结事件（onError/onCompleted）。

实现 `onXXX` 就比较简单了，都是同样的套路：

~~~ java
@Override
public void onNext(T t) {
    if (done) {
        return;
    }
    for (SubscriberState<T> innerState : subscribers) {
        innerState.onNext(t);
    }
}
 
@Override
public void onError(Throwable e) {
    if (done) {
        return;
    }
    error = e;
    done = true;
    for (SubscriberState<T> innerState : terminate()) {
        innerState.onError(e);
    }
}
 
@Override
public void onCompleted() {
    if (done) {
        return;
    }
    done = true;
    for (SubscriberState<T> innerState : terminate()) {
        innerState.onCompleted();
    }
}
~~~

只是简单地遍历当前的所有 Subscriber，向每个转发当前接收到的事件。

到目前为止，我们都只是把事件转发到另一个类（SubscriberState）中，现在是时候实现 SubscriberState，把事件发送给 child 了：

~~~ java
static final class SubscriberState<T> 
implements Producer, Subscription, Observer<T> {
    final Subscriber<? super T> child;                      // (1)
    final State<T> state;                                   // (2)
    final BackpressureStrategy strategy;                    // (3)
     
    final AtomicLong requested = new AtomicLong();          // (4)
     
    final AtomicInteger wip = new AtomicInteger();          // (5)
     
    volatile boolean unsubscribed;                          // (6)
 
    volatile boolean done;
    Throwable error;
 
    final Queue<T> queue;                                   // (7)
     
    public SubscriberState(
            Subscriber<? super T> child, State<T> state) {
        this.child = child;
        this.state = state;
        this.strategy = state.strategy;
        Queue<T> q = null;
        if (strategy == BackpressureStrategy.BUFFER) {      // (8)
            q = new SpscLinkedAtomicQueue<>();
        }
        this.queue = q;
    }
     
    @Override
    public void onNext(T t) {
        // TODO Auto-generated method stub
    }
     
    @Override
    public void onError(Throwable e) {
        // TODO Auto-generated method stub
    }
     
    @Override
    public void onCompleted() {
        // TODO Auto-generated method stub
    }
     
    @Override
    public void request(long n) {
        // TODO Auto-generated method stub
    }
     
    @Override
    public boolean isUnsubscribed() {
        return unsubscribed;
    }
     
    @Override
    public void unsubscribe() {
        // TODO Auto-generated method stub
    }
 
    void drain() {
        // TODO Auto-generated method stub
    }
}
~~~

1. 我们保持实际 Subscriber 的引用。
2. 当 child 取消订阅时，我们需要从 `State` 中移除 SubscriberState 自己。
3. 我们保存一个局部的 BackpressureStrategy，以避免每次都需要读外部类的成员。
4. 记录 child 的请求量。
5. 实现漏循环时需要一个 wip 变量。
6. 我们需要记录 child 是否已经调用了 `unsubscribe()`。
7. 如果 backpressure 策略是 BUFFER，那我们就需要临时保存过多的数据。
8. 最后，只有 backpressure 策略是 BUFFER 时，我们才会有一个队列实例。

接下来让我们一个一个实现上面的方法：

~~~ java
@Override
public void onNext(T t) {
    if (unsubscribed) {
        return;
    }
    switch (strategy) {
    case BUFFER:
        queue.offer(t);                               // (1)
        drain();
        break;
    case DROP: {
        long r = requested.get();                     // (2)
        if (r != 0L) {
            child.onNext(t);
            if (r != Long.MAX_VALUE) {
                requested.decrementAndGet();
            }
        }
        break;
    }
    case ERROR: {
        long r = requested.get();                     // (3)
        if (r != 0L) {
            child.onNext(t);
            if (r != Long.MAX_VALUE) {
                requested.decrementAndGet();
            }
        } else {
            unsubscribe();
            child.onError(
                new MissingBackpressureException());
        }
         
        break;
    }
    default:
    }
}
~~~

这个方法看起来有点复杂，但其实仅仅只是因为它处理了所有的策略，事实上还是很简单的：

1. 当处于 BUFFER 模式时，我们把数据加入到队列中，然后进入漏循环。
2. 在 DROP 模式时，我们检查请求量，如果有请求，就发送数据，如果不是无尽模式，就递减请求量，如果没有请求量，那就直接丢弃数据。
3. 在 ERROR 模式下，有请求的处理和 DROP 相同，没有请求时，我们就取消订阅，然后向 child 发出 `MissingBackpressureException`。

接下来是 `onError()` 和 `onCompleted()`，很直观：

~~~ java
@Override
public void onError(Throwable e) {
    if (unsubscribed) {
        return;
    }
    if (strategy == BackpressureStrategy.BUFFER) {
        error = e;
        done = true;
        drain();
    } else {
        child.onError(e);
    }
}
 
@Override
public void onCompleted() {
    if (unsubscribed) {
        return;
    }
    if (strategy == BackpressureStrategy.BUFFER) {
        done = true;
        drain();
    } else {
        child.onCompleted();
    }
}
~~~

剩下的三个方法，`request()`，`isUnsubscribed()` 和 `unsubscribed()` 看起来也应该很熟悉了：

~~~ java
@Override
public void request(long n) {
    if (n < 0) {
        throw new IllegalArgumentException();
    }
    if (n > 0) {
        BackpressureUtils.getAndAddRequest(requested, n);
        if (strategy == BackpressureStrategy.BUFFER) {
            drain();
        }
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
        state.remove(this);
        if (strategy == BackpressureStrategy.BUFFER) {
            if (wip.getAndIncrement() == 0) {
                queue.clear();
            }
        }
    }
}
~~~

取消订阅时，只有处于 BUFFER 模式时才需要进入漏循环，以及清空队列。

最后但也同样重要的，就是 `drain()` 了：

~~~ java
void drain() {
    if (wip.getAndIncrement() != 0) {
        return;
    }
     
    int missed = 1;
 
    Queue<> q = queue;
    Subscriber child = this.child;
 
    for (;;) {
 
        if (checkTerminated(done, q.isEmpty(), child)) {
            return;
        }
 
        long r = requested.get();
        boolean unbounded = r == Long.MAX_VALUE;
        long e = 0L;
 
        while (r != 0) {
            boolean d = done;
            T v = q.poll();
            boolean empty = v == null;
 
            if (checkTerminated(d, empty, child)) {
                return;
            }
 
            if (empty) {
                break;
            }
 
            child.onNext(v);
 
            r--;
            e--;
        }
 
        if (e != 0) {
            if (!unbounded) {
                requested.addAndGet(e);
            }
        }
 
        missed = wip.addAndGet(-missed);
        if (missed == 0) {
            return;
        }
    }
}
~~~

漏循环也和上文一模一样，毋庸赘言。

最后，`checkTerminated()` 还需要负责清理资源，让我们看看它的实现：

~~~ java
boolean checkTerminated(boolean done, 
        boolean empty, 
        Subscriber<? super T> child) {
    if (unsubscribed) {
        queue.clear();                     // (1)
        state.remove(this);
        return true;
    }
    if (done && empty) {
        unsubscribed = true;               // (2)
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

如果检测到当前已被取消订阅，那我们就清空队列，并把 SubscriberState 从 `State.subscribers` 数组中移除（1）。但是到达终结或者空的状态时，我们不需要移除自己（2），因为此时 Subject 已经处于终结状态，State 也已经不包含任何 Subscriber 了。

## 关于 `BehaviorSubject` 的一点啰嗦

`BehaviorSubject` 的行为介于 `PublishSubject` 和 `ReplaySubject` 之间，它在转发后续的事件之前，会重放此前最后一个 `onNext()` 事件。有人可能认为这可以通过容量为 1 的 `ReplaySubject` 实现，但实际上它们的终结状态行为不一样。容量为 1 的 `ReplaySubject` 会重放一个 `onNext()` 以及一个终结事件（`onError`/`onCompleted`），但 `BehaviorSubject` 不会重放 `onNext()`，只会发出终结事件。

从并发的角度来看，容量为 1 的 ReplaySubject 在处理并发 `subscribe()` 和 `onNext()` 时会更加复杂。因为规范要求，只要订阅调用之后，我们就不能错过任何事件，我们只能保存最后一个事件，然后在发出其他新的事件之前，把它发出去。

在 RxJava 1.x 的 `BehaviorSubject` 实现中，使用的方式是每个 Subscriber 一个锁，并且两种情况进行了不同的处理：首个和后续的 `onNext()`。当订阅发生时，订阅线程会尝试进入“首个”模式，从 Subject 中读出最后一个数据，然后发送出去。如果这时有一个并发的 `onNext()` 调用，`onNext()` 暂时会被阻塞住。当“首个”模式结束之后，就会进入“后续”模式，此后 `onNext()` 事件就会被立即转发了。

~~~ java
protected void emitNext(Object n, final NotificationLite<T> nl) {
    if (!fastPath) {
        synchronized (this) {
            first = false;
            if (emitting) {
                if (queue == null) {
                    queue = new ArrayList<Object>();
                }
                queue.add(n);
                return;
            }
        }
        fastPath = true;
    }
    nl.accept(actual, n);
}
 
protected void emitFirst(Object n, final NotificationLite<T> nl) {
    synchronized (this) {
        if (!first || emitting) {
            return;
        }
        first = false;
        emitting = n != null;
    }
    if (n != null) {
        emitLoop(null, n, nl);
    }
}
~~~

简单来说这是两个非对称的发射者循环：如果 `emitNext()` 赢得了竞争，那 `emitFirst()` 就不会运行了，那如果 `subscribe()` 和 `onNext()` 同时发生，谁说这个 `onNext()` 不是这个 Subscriber 订阅之前的最后一个 `onNext()` 呢？

此外，这种方式仍存在一个微妙的 bug。`emitFirst()` 有可能会把同样的数据发射两次。

在极端情况下，`onNext` 设置了最后的数据，然后 `emitFirst` 读到了这个数据，然后 `onNext` 尝试执行 `emitNext()`，这时 `emitNext` 发现有线程正在发送，所以就把数据加入到了队列中。而最终，`emitFirst` 在 `emitLoop` 中发现还有数据要发送，就会把数据取出来然后发送出去，这时数据就重复了。

解决办法比较复杂，大家可以在 [RxJava 2.x 的 BehaviorSubject](https://github.com/ReactiveX/RxJava/blob/2.x/src/main/java/io/reactivex/subjects/BehaviorSubject.java){:target="_blank"} 中看到。简单来说就是我们要为每个数据加上一个版本标签，锁住 `onNext()` 一小段时间，在这期间丢弃掉老的数据。这种方式很明显有一个缺点，就是我们在执行过程中多了一个阻塞代码块，而理论上来说，任何并发的 `subscribe` 都有可能阻塞住发射者循环。当然我们也可以实现一个无锁化版本，但这就要求我们每次 `onNext` 时都要分配一个不可变的数据以及索引变量了。

## 总结

在本文中，我演示了如何实现支持三种 backpressure 策略的 `PublishSubject`。这是我关于 Subject 的最后一篇文章了。

如果你看了 RxJava 1.x 的源码，就会发现标准的 Subject 并没有按照这样的方式实现，但 RxJava 2.x 是这样做的。这并不是因为什么错误，而是 2.x 的实现方式是基于 1.x 的教训重新设计的。

在下一个系列中，我将利用我们这里讲到的 Subject 的内部细节，然后展示如何实现 `ConnectableObservable`。
