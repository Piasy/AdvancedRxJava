---
layout: post
title: ConnectableObservables（三）
tags:
    - Operator
---

原文 [ConnectableObservables (part 3)](http://akarnokd.blogspot.com/2015/10/connectableobservables-part-3.html){:target="_blank"}。

## 介绍

在上一篇中，我们编写了一个当所有下游的 Subscriber 都请求了一定数量之后，就把数据发往下游的 ConnectableObservable，这让所有下游的 Subscriber 保持步调一致。

在本文中，我将具体展示如何编写一个重放的版本：ReplayConnectableObservable。它的类结构和 PublishConnectableObservable 类似，但请求处理的部分更加复杂。

## 无限重放 V.S. 有限重放

如果我们想要创建一个重放的操作符（或者 Subject），我们首先需要决定无限重放还是有限重放。无限重放意味着我们 connect 之后所有的事件都需要缓存起来，后续所有的 Subscriber 都将从最开始的事件重放。

有限重放则意味着随着时间的持续，或者缓存数据的增加，我们会丢弃掉老的数据，那么后来的 Subscriber 可能只能接收到较新的数据。

然而支持这两种模式的数据结构却大相径庭。无限重放可以使用任意列表型的数据结构，例如 `j.u.List`，或者使用链表（避免扩容时的拷贝）。有限重放则需要使用链表型数据结构，但这里不能使用 `j.u.LinkedList`，我们需要访问到每个单独的节点。

原因有二，1）我们需要随时知道当前缓冲的头部；2）我们需要处理 Subscriber 卡住，但却有请求还未处理的情况，这些 Subscriber 不能错过数据。

正确的数据结构是单向链表，每个节点保存实际数据。我们保存链表的头尾节点，头节点记录着开始重放的位置，尾节点则记录着上游数据加入的位置。

这种数据结构有两种效果：1）由于它是单向链接的，如果我们的头指针不再引用链表，并且所有的 Subscriber 也不再引用链表，那它就可以自动 GC 了；2）如果我们不移动头指针，那这个缓冲区就是无限容量的（尽管为了保存指针而多些开销）。

对于无限缓冲的模式，头指针的值将是 0，尾指针的值将是节点数量。

此外，每个 Subscriber 都需要记录它重放的位置，要么记录列表的下标，要么记录链表的节点。

由于我们要支持两种模式，而唯一的区别就是缓冲区的管理，那我们可以先定义出基本的缓冲区操作的接口：

~~~ java
interface ReplayBuffer<T> {
    void onNext(T value);
    void onError(Throwable e);
    void onCompleted();
    void replay(ReplayProducer<T> child);
}
~~~

接口的设计很直观，它可以接收各种事件，并且可以向指定的 Subscriber 重放事件。

## 无限重放缓冲区

现在让我们看看无限缓冲的实现：

~~~ java
static final class UnboundedReplayBuffer<T> implements ReplayBuffer<T> {
    final List<Object> values = new ArrayList<>();
    volatile int size;
    final NotificationLite<T> nl = NotificationLite.instance();
    @Override
    public void onNext(T value) {
        values.add(nl.next(value));
        size++;
    }
    @Override
    public void onError(Throwable e) {
        values.add(nl.error(e));
        size++;
    }
    @Override
    public void onCompleted() {
        values.add(nl.completed());
        size++;
    }
    @Override
    public void replay(ReplayProducer<T> child) {
        if (child.wip.getAndIncrement() != 0) {
            return;
        }
         
        int missed = 1;
         
        for (;;) {
             
            // implement
             
            missed = child.wip.addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        }
    }
}
~~~

我们把每个事件转化为一个 Notification 然后加入到列表中。递增 `volatile size` 的操作相当于一次发布（注意这里不需要原子自增操作，因为 onXXX 事件已经保证了串行发生），所以这个值表示着任何时刻可以遍历到缓冲区列表的位置（所有改变容量的操作都需要提交）。replay 函数剩下的部分，就是一个典型的队列漏了：单一线程将会进入循环，并做尽可能多的事情。让我们看看漏循环的部分：

~~~ java
// for (;;)
 
long r = child.requested.get();
boolean unbounded = r == Long.MAX_VALUE;
long e = 0;
int index = child.index;                    // (1)
 
while (r != 0L && index != size) {          // (2)
    if (child.isUnsubscribed()) {
        return;
    }
     
    Object v = values.get(index);           // (3)
     
    if (nl.accept(child.child, v)) {        // (4)
        return;
    }
     
    index++;
    r--;
    e--;                                    // (5)
}
 
if (e != 0L) {
    child.index = index;                    // (6)
    if (!unbounded) {
        child.requested.addAndGet(e);
    }
}
// missed = ...
~~~

看起来应该很熟悉了，让我们看看具体的过程：

1. 我们先取出这个 child 请求的数量，以及它的重放下标 `index`，我们记下它是不是请求的无尽模式，以及一个记录发射数量的计数器。
2. 我们只在 child 发出了请求，而且还有数据可以重放时，才进行重放。
3. 如果需要重放，那我们就取出 index 处的数据。
4. `NotificationLite.accept` 会把 Notification 对象转化为对 child 的 onXXX 调用，并且如果是终止事件，就会返回 true。
5. 我们递增重放下标，递减请求数量，递减发射计数。递减发射计数看起来很奇怪，但它能节省我们在（6）处的一次取负操作；
6. 最后，如果我们发出了数据，就更新 child 的重放位置下标，如果 child 的请求量不是无限大，那就从中减去发射量；

## 有限重放缓冲区

管理一个有限的缓冲区稍微复杂一些，这里我演示如何实现一个基于数量的有限缓冲区，大家可以基于此实现自己的缓冲区控制逻辑。首先我们需要一个 `Node` 类来容纳实际的数据，以及指向下一个节点的引用。

~~~ java
static final class Node {
    final Object value;
    final long id;
    volatile Node next;
    public Node(Object value, long id) {
        this.value = value;
        this.id = id;
    }
}
~~~

Node 包括了实际数据，指向下一节点的引用，以及一个 id 成员，id 将用于后面的请求处理。

现在让我们看看 `BoundedReplayBuffer` 的实现：

~~~ java
static final class BoundedReplayBuffer<T>
implements ReplayBuffer<T> {
     
    final NotificationLite<T> nl = 
            NotificationLite.instance();
     
    volatile Node head;                            // (1)
    Node tail;
    int size;                                      // (2)
    final int maxSize;
    long id;
     
     
    public BoundedReplayBuffer(int maxSize) {      // (3)
        this.maxSize = maxSize;
        tail = new Node(null, 0);
        head = tail;
    }
     
    void add(Object value) {                       // (4)
        Node n = new Node(value, ++id);
        Node t = tail;
        tail = n;
        t.next = n;
    }
     
    @Override
    public void onNext(T value) {
        add(nl.next(value));
        if (size == maxSize) {                     // (5)
            Node h = head;
            head = h.next;
        } else {
            size++;
        }
    }
     
    @Override
    public void onError(Throwable e) {             // (6)
        add(nl.error(e));
    }
     
    @Override
    public void onCompleted() {
        add(nl.completed());
    }
     
    @Override
    public void replay(ReplayProducer<T> child) {  // (7)
        if (child.wip.getAndIncrement() != 0) {
            return;
        }
         
        int missed = 1;
 
        for (;;) {
             
            // implement
             
            missed = child.wip.addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        }
    }
}
~~~

有限缓冲区就需要考虑更多状态了：

1. 我们需要 head 和 tail 指向链表的首尾节点。head 必须是 volatile 的，因为 Subscriber 订阅时，我们需要访问它。tail 则只会被上游事件发出的线程修改（不一定是同一个线程，但已经保证串行访问，也就保证了可见性），不会被 Subscriber 访问，所以无需 volatile。
2. 由于我们需要限制重放的数量，所以我们需要记录缓冲区的大小（避免每次遍历链表），以及最大限制。除此之外，还有一个递增的唯一 id，用于后面的请求管理。
3. 在构造函数中，我们创建一个空节点，并把它赋值给 head 和 tail，这是典型的链表套路。这样做有两个小的不足：a）每次取缓冲区的首元素都需要用 `head.next.value`，略不直接；b）由于我们会在（5）处后移 head，我们会多占用一个数据，无法被 GC。也就是说，`replay(5)` 会保留 6 个元素。RxJava 的 replay() 和 ReplaySubject 都有这样的问题。_如果你一定想把这个多余的保留元素去掉，可以使用引用计数技术，但这会为每一次操作带来开销，包括把元素加入到缓冲区和重放元素。_
4. 我们通过 add 函数把新的事件加入到链表中。这个操作很直观：用一个唯一的 id 创建一个新的节点，把它设置为新的 tail，并把老的 `tail.next` 指向它。顺序很重要，因为 next 是 volatile 的，为它赋值相当于一次内存操作的发布，我们要最后进行。
5. 当 onNext 事件到达时，我们把它加到链表中，并检查是否已经超出缓冲区大小。如果未超出，我们直接把 size 加一。如果已超过，直接把 head 往后移动一个元素即可。由于我们链表的结构保证了链表至少有一个元素（add 函数保证的），所以新的 head 不会是 null。
6. 由于终止事件通常不计入缓冲区大小，所以我们直接把它们加入到链表中即可。
7. 最后，还是那个漏循环。

现在我们看看漏循环的实现：

~~~ java
// for (;;) {
long r = child.requested.get();
boolean unbounded = r == Long.MAX_VALUE;
long e = 0;
Node index = child.node;
 
if (index == null) {                       // (1)
    index = head;
    child.node = index;
     
    child.addTotalRequested(index.id);     // (2)
}
 
while (r != 0L && index.next != null) {    // (3)
    if (child.isUnsubscribed()) {
        return;
    }
     
    Object v = index.value;
     
    if (nl.accept(child.child, v)) {
        return;
    }
     
    index = index.next;                    // (4)
    r--;
    e--;
}
 
if (e != 0L) {
    child.node = index;
    if (unbounded) {
        child.requested.addAndGet(e);
    }
}
// missed = ...
~~~

毫无疑问，我们用了和 UnboundedReplayBuffer 一样的结构，但有以下几点差异：

1. 由于节点是引用，默认值是 null，所以第一次调用 replay 的时候，我们需要把当前缓冲区的头节点保存到 child 中。
2. `addTotalRequested` 函数会获取节点的 id，id 的作用将在后面的请求处理部分讲解。
3. 为了知道我们是否已经把缓冲区重放完毕，我们需要检查 `index.next`。
4. 如果 `index.next` 不为 null，那我们就可以重放一个事件，并把 index 后移一个节点。

## 请求处理

到目前为止，应该都没有什么复杂的地方，只是介绍了 ReplayProducer 的结构，以及两中缓冲区的实现。

在上一篇中我们已经提到，通常请求处理有两种方式：步调一致，或者取最大值。步调一致的方案很适合 PublishConnectableObservable。

现在我们考虑一下步调一致的方案是否适用于 replay。如果我们要实现无限缓冲的 replay，那是否步调一致也无关紧要，因为不管数据是什么时候请求来的，我们都要保存起来；每个 Subscriber 都会收到它请求数量的数据，相互之间并不影响。但如果有一个 Subscriber 能接收所有的数据，为什么不把它们都请求过来呢？

而如果我们想要实现有限缓冲区，由于 Subscriber 可以在任何时间订阅和取消订阅，它们在订阅时 BoundedReplaySubject 的 id 成员值都不一样，而每个 Subscriber 都会基于订阅时的 id 值进行请求。所以这时就无法有效地定义最小请求量了，如果基于一个较小的 id 请求 5 个数据，和基于一个更大的 id 请求 2 个数据，它们失去了可比性。

基于以上原因，我们实现 replay 时将取所有 Subscriber 请求的最大数量发给上游。

然而我们依然面临请求量不具备可比性的问题，因为 Subscriber 订阅的时间不同。这时我们就要用上那个唯一的 id，以及下面的方式了：我们会记下总的请求数量和相对请求数量。当有 Subscriber 发出请求时，我们把请求数量加到它的总请求量 `totalRequested` 中（`ReplayPublisher.totalRequested`），并检查这个数量是否超过我们发往上游的总请求数量。如果超过，我们就把差值发往上游。

在这样的机制下，唯一的 id 对后来的 Subscriber 就很有用了。如果没有这个 id，那后来的 Subscriber 的请求计数就可能不会超过发往上游的请求数。例如，`range(1, 10).replay(1)`，一个 Subscriber 请求了两个数据并收到之后，又新来了一个 Subscriber，并且也请求了两个数据。显然后来的应该收到 `(2, 3)`，但由于 2 并没有超过发往上游的总请求，所以操作符并不会向上游请求新的数据，这样就只会收到 `(2)` 这一个数据了。解决方案就是为每个数据记录索引，当前 Node 首次被 Subscriber 记下时，把 index 作为这个 Subscriber 的总请求量，就好像这个 Subscriber 一直都在这里，但只是忽略了此前的所有数据。

注意：这个属性刚刚被发现，因此，RxJava 还不能正常工作。[PR #3454](https://github.com/ReactiveX/RxJava/pull/3454) 解决了 1.x 的这个问题，2.x 我之后会新提一个 PR。

为了让这个问题更加清晰，我们来看看 ReplayProducer 的实现：

~~~ java
static final class ReplayProducer<T> 
implements Producer, Subscription {
    int index;
    Node node;                                    // (1)
    final Subscriber<? super T> child;
    final AtomicLong requested;
    final AtomicInteger wip;
    final AtomicLong totalRequested;
    final AtomicBoolean once;                     // (2)
 
    Connection<T> connection;
 
    public ReplayProducer(
            Subscriber<? super T> child) {
        this.child = child;
        this.requested = new AtomicLong();
        this.totalRequested = new AtomicLong();
        this.wip = new AtomicInteger();
        this.once = new AtomicBoolean();
    }
 
    @Override
    public void request(long n) {
        if (n > 0) {
            BackpressureUtils
            .getAndAddRequest(requested, n);
            BackpressureUtils
            .getAndAddRequest(totalRequested, n); // (3)
 
            connection.manageRequests();          // (4)
        }
    }
 
    @Override
    public boolean isUnsubscribed() {
        return once.get();
    }
 
    @Override
    public void unsubscribe() {
        if (once.compareAndSet(false, true)) {
            connection.remove(this);             // (5)
        }
    }
 
    void addTotalRequested(long n) {             // (6)
        if (n > 0) {
            BackpressureUtils
            .getAndAddRequest(totalRequested, n);
        }
    }
}
~~~

这个类的目的是被设置到 Subscriber 上去，处理请求以及取消订阅：

1. 由于我们想要在两种缓冲模式中用同一个类，所以我们同时保存了 `index` 和 `node` 成员。
2. 接下来是其他正常需要的成员：child Subscriber，队列漏用到的 wip 计数器，当前的请求量计数器，以及一个 AtomicBoolean 成员标记是否已经取消订阅。除此之外还有一个成员记录了总的请求量，以帮助我们决定向上游发出多少请求量。
3. 当下游请求时，我们更新两个请求计数器，利用 `BackpressureUtils` 我们把请求计数限制在 `Long.MAX_VALUE` 以内。
4. 更新了请求计数器之后，我们触发请求管理函数，决定是否向上游发出请求。
5. 当下游取消订阅时，我们需要把 ReplayProducer 从 Connection 的 ReplayProducer 数组中移除。
6. 最后，有限缓冲区的重放需要在发射数据之前更新总请求计数，以便请求处理机制对后来的 Subscriber 也能有效。

在看 manageRequests() 的实现之前，我们先看一下 Connection 类的结构（和 PublishConnectableObservable 的类似）：

~~~ java
@SuppressWarnings({ "unchecked", "rawtypes" })
static final class Connection<T> implements Observer<T> {
 
    final AtomicReference<ReplayProducer<T>[]> subscribers;
    final State<T> state;
    final AtomicBoolean connected;
    final AtomicInteger wip;
 
    final SourceSubscriber parent;
 
    final ReplayBuffer<T> buffer;                        // (1)
 
    static final ReplayProducer[] EMPTY = 
        new ReplayProducer[0];
 
    static final ReplayProducer[] TERMINATED = 
        new ReplayProducer[0];
     
    long maxChildRequested;                              // (2)
    long maxUpstreamRequested;
 
    public Connection(State<T> state, int maxSize) {
        this.state = state;
        this.wip = new AtomicInteger();
        this.subscribers = new AtomicReference<>(EMPTY);
        this.connected = new AtomicBoolean();
        this.parent = createParent();
         
        ReplayBuffer b;                                 // (3)
        if (maxSize == Integer.MAX_VALUE) {
            b = new UnboundedReplayBuffer<>();
        } else {
            b = new BoundedReplayBuffer<>(maxSize);
        }
        this.buffer = b;
    }
 
    SourceSubscriber createParent() {                   // (4)
        SourceSubscriber parent = 
            new SourceSubscriber<>(this);
 
        parent.add(Subscriptions.create(() -> {
            switch (state.strategy) {
            case SEND_COMPLETED:
                onCompleted();
                break;
            case SEND_ERROR:
                onError(new CancellationException(
                    "Disconnected"));
                break;
            default:
                parent.unsubscribe();
                subscribers.getAndSet(TERMINATED);
            }
        }));
 
        return parent; 
    }
 
    boolean add(ReplayProducer<T> producer) {
        // omitted
    }
 
    void remove(ReplayProducer<T> producer) {
        // omitted 
    }
 
    void onConnect(
    Action1<? super Subscription> disconnect) {
        // omitted
    }
 
    @Override
    public void onNext(T t) {                          // (5)
        ReplayBuffer<T> buffer = this.buffer;
        buffer.onNext(t);
        ReplayProducer<T>[] a = subscribers
            .get();
        for (ReplayProducer<T> rp : a) {
            buffer.replay(rp);
        }
    }
 
    @Override
    public void onError(Throwable e) {
        ReplayBuffer<T> buffer = this.buffer;
        buffer.onError(e);
        ReplayProducer<T>[] a = subscribers
            .getAndSet(TERMINATED);
        for (ReplayProducer<T> rp : a) {
            buffer.replay(rp);
        }
    }
 
    @Override
    public void onCompleted() {
        ReplayBuffer<T> buffer = this.buffer;
        buffer.onCompleted();
        ReplayProducer<T>[] a = subscribers
            .getAndSet(TERMINATED);
        for (ReplayProducer<T> rp : a) {
            buffer.replay(rp);
        }
    }
 
    void manageRequests() {                           // (6)
        if (wip.getAndIncrement() != 0) {
            return;
        }
         
        int missed = 1;
         
        for (;;) {
 
            // implement            
             
            missed = wip.addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        }
    }
}
~~~

这个类看起来和 PublishConnectableObservable.Connect 很相似，所以我忽略了完全一样的函数，其他的部分解释如下：

1. 这里我们声明的是 ReplayBuffer 成员，而不是有界的队列。
2. 我们需要记录下游的最大请求量，以及发往上游的请求总量。因为我们不知道上游的 Producer 何时被设置，所以我们必须在它到来之前记下所有的请求量。
3. 我们把 `Integer.MAX_VALUE` 作为无尽缓冲的标志。
4. createParent 函数略有不同，这里我们无需 disconnected 标记，而是可以直接把上游取消订阅了。其他 add，remove 和 onConnect 函数和[上一篇文章](/AdvancedRxJava/2017/03/03/connectableobservables-part-2/index.html)完全一样。
5. onXXX 函数有同样的模式：调用 ReplayBuffer 相应的函数，以及对每个 ReplayProducer 调用 replay 函数。注意终止事件会用原子操作把 TERMINATED 设置给 ReplayProducer 数组，这样后来的 Subscriber 就得等到下一个 Connection 对象了。
6. 最后我们需要管理请求，由于各个 Subscriber 的请求可能并发到来，所以我们需要串行化。由于我们需要计算请求的最大值，所以非阻塞的同步方式可以满足需求。这个函数将在上游的 Producer 到达时调用，以及每个下游发出请求时调用。

现在让我们看看请求管理的代码：

~~~ java
// for (;;) {
 
ReplayProducer<T>[] a = subscribers.get();
 
if (a == TERMINATED) {
    return;
}
 
long ri = maxChildRequested;
long maxTotalRequests = ri;                 // (1)
 
for (ReplayProducer<T> rp : a) {
    maxTotalRequests = Math.max(
        maxTotalRequests, 
        rp.totalRequested.get());
}
 
long ur = maxUpstreamRequested;
Producer p = parent.producer;
 
long diff = maxTotalRequests - ri;          // (2)
if (diff != 0) {
    maxChildRequested = maxTotalRequests;
    if (p != null) {                        // (3)
        if (ur != 0L) {
            maxUpstreamRequested = 0L;
            p.request(ur + diff);           // (4)
        } else {
            p.request(diff);
        }
    } else {
        long u = ur + diff;
        if (u < 0) {
            u = Long.MAX_VALUE;
        }
        maxUpstreamRequested = u;           // (5)
    }
} else
if (ur != 0L && p != null) {                // (6)
    maxUpstreamRequested = 0L;
    p.request(ur);
}
 
// missed = ...
~~~

让我们看看它的工作原理：

1. 获取了当前的 Subscriber 数组，以及检查了终止状态之后，我们计算下游请求量（以及之前的最大值 maxChildRequested）的最大值。
2. 我们计算之前的最大值和当前最大值的差，如果差不为 0，我们就更新 maxChildRequested；
3. 此时上游的 Producer 可能仍未到达。
4. 如果上游的 Producer 已经设置，那我们就把此前积累的请求，以及这次的差值，一起发往上游。
5. 否则我们只能继续积累请求。
6. 如果差值为零，上游 Producer 已经设置，且之前有积累的请求，我们仍需要发往上游。而且这里和（4）处，都需要清空积累的请求量，后续的处理，只需要把差值发往上游即可。

换言之，我们就是统计每个下游 Subscriber 的请求量，并基于此向上游发出请求。

大家可能已经注意到，如果有大量的 Subscriber 不停地请求，那这个请求管理的开销会变得很大。实际上，我们每次只需要考虑发出了请求的 Subscriber。这样我们的发射者循环或者队列漏需要接受一个参数，循环中只考虑发出了请求的 Subscriber。这种模式下，如果只有一个 Subscriber 发出了请求，那就只有这一个 Subscriber 会被处理。

这里我们还有一种情况需要考虑：当上游的 Producer 被设置时，我们需要处理所有 Subscriber 的请求。为此，我们需要让 Connection 类增加几个成员：

~~~ java
List<ReplayProducer<T>> coordinationQueue;
boolean coordinateAll;
boolean emitting;
boolean missed;
~~~

大家可能已经猜到了这里要使用的方式了：发射者循环。我们可以去掉 wip 计数器，并把它替换为 `emitting` 和 `missed` 变量。

~~~ java
void manageRequests(ReplayProducer<T> inner) {
    synchronized (this) {                               // (1)
        if (emitting) {
            if (inner != null) {
                List<ReplayProducer<T>> q = 
                    coordinationQueue;
                if (q == null) {
                    q = new ArrayList<>();
                    coordinationQueue = q;
                }
                q.add(inner);
            } else {
                coordinateAll = true;
            }
            missed = true;
            return;
        }
        emitting = true;
    }
     
    long ri = maxChildRequested;
    long maxTotalRequested;
     
    if (inner != null) {                                // (2)
        maxTotalRequested = Math.max(
            ri, inner.totalRequested.get());
    } else {
        maxTotalRequested = ri;
 
        @SuppressWarnings("unchecked")
        ReplayProducer<T>[] a = producers.get();
        for (ReplayProducer<T> rp : a) {
            maxTotalRequested = Math.max(
                maxTotalRequested, rp.totalRequested.get());
        }
         
    }
    makeRequest(maxTotalRequested, ri);
     
    for (;;) {
        if (isUnsubscribed()) {
            return;
        }
         
        List<ReplayProducer<T>> q;
        boolean all;
        synchronized (this) {                           // (3)
            if (!missed) {
                emitting = false;
                return;
            }
            missed = false;
            q = coordinationQueue;
            coordinationQueue = null;
            all = coordinateAll;
            coordinateAll = false;
        }
         
        ri = maxChildRequested;                         // (4)
        maxTotalRequested = ri;
 
        if (q != null) {
            for (ReplayProducer<T> rp : q) {
                maxTotalRequested = Math.max(
                maxTotalRequested, rp.totalRequested.get());
            }
        } 
         
        if (all) {
            @SuppressWarnings("unchecked")
            ReplayProducer<T>[] a = producers.get();
            for (ReplayProducer<T> rp : a) {
                maxTotalRequested = Math.max(
                maxTotalRequested, rp.totalRequested.get());
            }
        }
         
        makeRequest(maxTotalRequested, ri);
    }
}
~~~

它的工作机制如下：

1. 首先我们尝试进入发射者循环。如果失败，且参数是 null，那我们就把 coordinateAll 置为 true，这会触发对所有 Subscriber 的处理。如果参数不为 null，那我们就把它加入到等待队列中。最后返回。
2. 如果当前线程进入了发射者循环，那我们就计算最大请求量，如果参数不为 null，那只需考虑传入的 ReplayProducer，否则需要考虑所有的 ReplayProducer。
3. 接下来就是发射者循环的主体了，我们检查是否有待处理的请求，然后取出待处理的 ReplayProducer，以及全量处理的标记。
4. 最后我们按照各种情况计算出最大请求量，考虑待处理队列中的 ReplayProducer，或者所有已知的 ReplayProducer。注意两种情况都需要考虑，因为这两者并没有包含关系。

最后，向上游发出请求的代码被封装进一个函数中：

~~~ java
void makeRequest(long maxTotalRequests, long previousTotalRequests) {
    long ur = maxUpstreamRequested;
    Producer p = producer;
 
    long diff = maxTotalRequests - previousTotalRequests;
    if (diff != 0) {
        maxChildRequested = maxTotalRequests;
        if (p != null) {
            if (ur != 0L) {
                maxUpstreamRequested = 0L;
                p.request(ur + diff);
            } else {
                p.request(diff);
            }
        } else {
            long u = ur + diff;
            if (u < 0) {
                u = Long.MAX_VALUE;
            }
            maxUpstreamRequested = u;
        }
    } else
    if (ur != 0L && p != null) {
        maxUpstreamRequested = 0L;
        // fire the accumulated requests
        p.request(ur);
    }
}
~~~

它的逻辑和前面每次都考虑所有 Subscriber 的 manageRequests() 函数一样。

## ReplayConnectableObservable

本文最后剩下的就是 SourceSubscriber 和 ReplayConnectableObservable 了。

由于我们需要访问上游的 Producer，所以我们用 SourceSubscriber 来把它保存起来，并且在设置时立即使用。这里我们不能使用 `Subscriber.request()`，有两个考虑：a）request() 的调用并不会在 Producer 到来之前积累请求；b）我们无法知道 Producer 是否到达。

~~~ java
static final class SourceSubscriber<T> 
extends Subscriber<T> {
    final Connection<T> connection;
     
    volatile Producer producer;
     
    public SourceSubscriber(Connection<T> connection) {
        this.connection = connection;
    }
 
    @Override
    public void onNext(T t) {
        connection.onNext(t);
    }
 
    @Override
    public void onError(Throwable e) {
        connection.onError(e);
    }
 
    @Override
    public void onCompleted() {
        connection.onCompleted();
    }
 
    @Override
    public void setProducer(Producer p) {
        producer = p;
        connection.manageRequests();
    }
}
~~~

没有什么特殊之处：我们把所有的事件都委托到 Connection 对象上。注意 `connection.manageRquests()` 会触发请求处理逻辑，把所有积累的请求量 maxUpstreamRequested 发往上游。如果我们要使用性能优化过的版本，就调用 `manageRequests(null)`。

State 类也稍有改动，因为我们需要考虑是否使用无限缓冲区，以及在新的 Subscriber 订阅时，应该开始重放。

~~~ java
static final class State<T> implements OnSubscribe<T> {
    final DisconnectStrategy strategy;
    final Observable<T> source;
    final int maxSize;                                     // (1)
 
    final AtomicReference<Connection<T>> connection;
 
    public State(DisconnectStrategy strategy, 
    Observable<T> source, int maxSize) {
        this.strategy = strategy;
        this.source = source;
        this.maxSize = maxSize;
        this.connection = new AtomicReference<>(
            new Connection<>(this, maxSize));
    }
 
    @Override
    public void call(Subscriber<? super T> s) {
        // implement
        ReplayProducer<T> pp = new ReplayProducer<>(s);
 
        for (;;) {
            Connection<T> curr = this.connection.get();
 
            pp.connection = curr;
 
            if (curr.add(pp)) {
                if (pp.isUnsubscribed()) {
                    curr.remove(pp);
                } else {
                    curr.buffer.replay(pp);               // (2)
 
                    s.add(pp);
                    s.setProducer(pp);
                }
                 
                break;
            }
        }
    }
 
    public void connect(
    Action1<? super Subscription> disconnect) {
        // same as before
    }
 
    public void replaceConnection(Connection<T> conn) {   // (3)
        Connection<T> next = 
            new Connection<>(this, maxSize);
        connection.compareAndSet(conn, next);
    }
}
~~~

改动如下：

1. 我们需要保存 maxSize 参数，因为重新建立连接时，我们需要重新创建 ReplayBuffer 对象。
2. 创建了 ReplayProducer 对象后，我们首先尝试把它加入到 Connection 对象中，如果加入成功，我们就进行一次重放。由于 ReplayProducer 此时的请求量是 0，所以这次调用不会重放任何数据。这里我们主要是为了把缓冲区链表的头节点保存到 ReplayProducer 中（如果是有限缓冲区模式），并确保它的起始总请求计数是正确的。只有在这一步设置完成之后，它才被作为取消订阅的回调以及 Producer 被设置给下游。
3. 注意现在 Connection 也需要一个 maxSize 参数了。

_注意，（2）中描述的顺序只有在上面我实现的版本中才能工作，即只有在请求之后才会重放终止事件，尽管这不是终止事件的必要要求或者预期行为，然而即便不用上面的版本也不会有大问题，因为绝大多数 Subscriber 都会发出请求。_

最后，我们还是需要工厂方法来创建 ReplayConnectableObservable 实例：

~~~ java
public static <T> ReplayConnectableObservable<T> createUnbounded(
        Observable<T> source, 
        DisconnectStrategy strategy) {
    return createBounded(source, strategy, Integer.MAX_VALUE);
}
 
public static <T> ReplayConnectableObservable<T> createBounded(
        Observable<T> source, 
        DisconnectStrategy strategy, int maxSize) {
    State<T> state = new State<>(strategy, source, maxSize);
    return new ReplayConnectableObservable<>(state);
}
~~~

## 总结

在本文中，我详细讲解了 replay 版本的 ConnectableObservable 的内部工作原理，它既能支持无限缓冲，也能支持有限缓冲。它的复杂度比上一篇的 PublishConnectableObservable 再升一级，如果你理解了上一篇文章，那它看起来就不是一个太大的台阶。复杂度来自于缓冲区的管理，以及选择最大请求量的策略。

在下一篇文章中，我将讲解如何把这样的 ConnectableObservable 优化为处理了请求的 Subject。在 RxJava 2.0 中，请求处理对 Subject 和 Reactive-Stream Processor 来说可能将是强制要求，当然这取决于相关的讨论结果。（_最终 RxJava 2.x 的实现结果可以参考 [wiki](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0#subjects-and-processors)_）
