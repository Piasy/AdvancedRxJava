---
layout: post
title: ConnectableObservables（一）
tags:
    - Operator
---

原文 [ConnectableObservables (part 1)](http://akarnokd.blogspot.com/2015/10/connectableobservables-part-1.html){:target="_blank"}。

## 介绍

前面我们已经讲解了[怎么创建 cold Observable，例如 range](/AdvancedRxJava/2016/05/18/operator-concurrency-primitives-3/){:target="_blank"}，以及[怎么创建 hot Observable，例如 UnicastSubject](/AdvancedRxJava/2016/10/04/subjects-part-2/){:target="_blank"}，但还没有涉及这两者之间的转换。

显然，由于 Subject 也是 Observable，我们只能把 Subject 订阅到一个 cold Observable 上，然后让所有的 Subscriber 都订阅到这个 Subject 上。

但我们为什么要这么做？这样做的主要好处是，cold Observable 被订阅时的“副作用”只会发生一次（后续 Subscriber 都不是直接订阅到源 Observable）。这意味着我们可以把同一个数据流用作不同的目的，避免出现多个相互独立的数据流。

例如你想要考察同一个数据流里面的相邻元素，你可以把它发布出去（publish），然后订阅多次，分别使用不同部分的数据，再把结果合并起来：

~~~ java
Observable<Integer> source = Observable.range(1, 10);
 
ConnectableObservable<Integer> published = source.publish();
 
Observable<Integer> first = published;
Observable<Integer> second = published.skip(1);
 
Observable<String> both = first.zipWith(second, 
    (a, b) -> a + "+" + b);
 
both.subscribe(System.out::println);
 
published.connect();
~~~

现在让我们看看 `ConnectableObservable` 的功能。

## ConnectableObservable 的功能要求

ConnectableObservable 是继承自 Observable 的抽象类，它需要额外实现一个方法。

由于继承了 Observable，它就和 Subject 一样面临了一个问题：在构造函数中，调用 super 之前不能调用其他函数。所以我们也需要用一个静态工厂方法和一个中间的状态对象。

第二个要求也来自于 Observable，subscribe 函数必须是线程安全的，而且不管订阅发生在 ConnectableObservable “运行”之前、之中还是之后。

额外的这个方法签名是 `connect(Action1<Subscription> s)` 而不是 `connect()`，因为要考虑到同步取消订阅的需求。那什么时候会有这种需求？有两种情况下，这一功能至关重要，一种是外部使用者的需求，另一种是一些内部操作符的需求。

问题在于如果连接到一个 cold 且同步的 Observable，那 connect 调用就会导致这个 Observable 开始发射数据直到结束（如果这个 Observable 是无尽的，那就不会结束，那 connect 函数就无法返回了）。被连接的 Observable 不是无尽时，如果 connect 时还没有 Subscriber 订阅，那这些数据就永远丢失了（因为被连接的 Observable 已经终止了）。而如果被连接的 Observable 是无尽的，那我们也没有机会取消订阅，因为 connect 函数不会返回了。

第二种情况在广播操作符中经常出现，例如 `publish(Func1)` 和 `replay(Func1)`。它们会创建一个 ConnectableObservable，调用传入的 Func1，然后返回一个普通的 Observable，并在这个 Observable 被订阅时，连接（connect）这个 ConnectableObservable。如果上游是一个同步的 Observable，而我们又只想从返回的 Observable 接收少量的数据，那这时就无法返回 Subscription，整个数据流就会一直运行下去了。

解决办法就是再写一个接收 callback 的版本，在 connect 之前先调用回调，把 Subscription 传过去，这样就有可能取消订阅了。

最后，连接和断开连接操作应该是幂等的。这意味着多次连接同一个流不会额外做任何事情，就像取消订阅同一个流多次，只会取消一次一样。关于取消订阅有一点需要注意，如果我们取消订阅了一个一个流，然后再次连接上去，那第一次连接时的 Subscription 不应该对第二次连接的状态产生影响。

总结一下，ConnectableObservable 需要满足一下要求：

+ 在任何时机、从任何线程订阅它，都应该是线程安全的；
+ 允许在任何时机、从任何线程同步地取消订阅；
+ connect 和 disconnect 都应该是幂等的；

## 简单实现

基于我们现在对 ConnectableObservable 和 Subject 的理解，利用 Subject 很容易实现 ConnectableObservable。让我们看看这一实现：

~~~ java
public final class Multicast<T>
extends ConnectableObservable<T> {
     
    final Observable<T> source;
    final Subject<T, T> subject;
     
    final AtomicReference<Subscription> subscription;         // (1)
     
    public Multicast(Observable<T> source, 
            Subject<T, T> subject) {
        super(s -> {
            subject.subscribe(s);                             // (2)
        });
        this.source = source;
        this.subject = subject;
        this.subscription = new AtomicReference<>();
    }
     
    @Override
    public void connect(
        Action1<? super Subscription> connection) {
        // implement
    }
}
~~~

到目前为止没有任何特殊之处，我们接收一个 Observable，以及一个 Subject，并把当前连接的引用保存在 AtomicReference 中（1）。这里 OnSubscribe 的逻辑很简单，并不需要像 UnicastSubject 一样搞一个工厂方法：对每一个新来的 Subscriber，我们直接订阅到 Subject 上（2）。

`connect()` 的实现稍微有趣一点，但也不复杂：

~~~ java
@Override
public void connect(Action1<? super Subscription> connection) {
    for (;;) {
        Subscription s = subscription.get();                   // (1)
        if (s != null) {
            connection.call(s);                                // (2)
            return;
        }
         
        Subscriber<T> subscriber = new Subscriber<T>() {       // (3)
            @Override
            public void onNext(T t) {
                subject.onNext(t);
            }
             
            @Override
            public void onError(Throwable e) {
                subject.onError(e);
            }
             
            @Override
            public void onCompleted() {
                subject.onCompleted();
            }
        };
         
        subscriber.add(Subscriptions.create(() -> {            // (4)
            subscription.set(null);
        }));
         
        if (subscription.compareAndSet(null, subscriber)) {    // (5)
            connection.call(subscriber);                       // (6)
             
            source.subscribe(subscriber);                      // (7)
             
            return;
        }
    }
}
~~~

这个实现就是一个简单的 CAS 循环：

1. 我们把当前连接的 Subscription 保存在 `subscription` 成员中，如果它不为 null，则说明  现在有一个活跃的连接。
2. 如果当前有活跃的连接，我们直接调用传入的 action。
3. 否则，说明当前没有活跃的连接，我们就建立一个。大家可能会问，为什么不直接用 Subject 去订阅 Observable？因为我们需要保证同步取消订阅：调用 `subscribe()` 返回 Subscription 时可能太晚（如果上游是同步的，要么 subscribe 返回时上游已经终止，要么永远不会返回），所以我们在订阅到上游之前就需要一个 Subscription。我们新创建的 Subscriber 会转发上游的事件，同时让我们可以取消订阅（Subscriber 继承自 Subscription）。
4. 如果连接（我们创建的 Subscriber）被取消订阅，我们需要把 `subscription` 成员重置为 null，使得之后可以继续连接。
5. 为了保证幂等性，我们利用 CAS 把 `subscription` 从 null 赋值。如果 CAS 失败，说明有并发线程连接成功，那我们会在（1）处退出。
6. 如果 CAS 成功，我们先调用传入的 action，这就使得调用方可以同步取消订阅了。
7. 最后，我们把 Subscriber 订阅到 Observable，然后返回。

## 简单实现的缺陷

上面简单的实现方案看起来可以用，但存在一些缺陷。

_注：在本文中，我希望教会读者如何发现操作符的 bug 和不足，所以有些例子并不是在一开始就把事情处理得很完美。_

第一个问题就是，如果上游终止了，重置 `subscription` 的代码可能在之后的某个时刻执行（通过 `SafeSubscriber`），或者根本不会执行。解决办法就是在 onError 和 onCompleted 中也进行重置，但都需要有条件地重置，因为它有可能已经被重置过了（正常的取消订阅和终止事件会发生竞争）。简单来说，重置的代码应该是这样子的：

~~~ java
    // ...
    @Override
    public void onError(Throwable e) {
        subject.onError(e);
        subscription.compareAndSet(this, null);
    }
 
    @Override
    public void onCompleted() {
        subject.onCompleted();
        subscription.compareAndSet(this, null);
    }
    // ...
 
subscriber.add(Subscriptions.create(() -> {
    subscription.compareAndSet(subscriber, null);
}));
~~~

这三种情况下，只有当 `subscription` 仍是之前的 Subscriber 时，我们才进行重置。这样在重新连接之后的终止事件，或者对老的连接调用取消订阅，才不会影响新的连接。

第二个问题是，如果上游进入了终止状态，Subject 也会进入终止状态。之后的连接都会直接被取消订阅，Subscriber 也只会收到终止事件（这是 RxJava 标准 Subject 的特性）。

这通常不是业务逻辑想要的效果，因此我们需要修改 Multicast 的参数，每次重新连接时都传入一个新的 Subject。我会在下一节讲解怎么实现，但在此之前，我们先看看当前实现的最后一个问题。

最后一个问题就是没有任何请求的协调：Subscriber 和 Subject 都会运行在无尽模式下，忽略任何 backpressure 请求。由于标准的 RxJava 1.x Subject 不支持 backpressure，所以我们很可能会在下游收到 MissingBackpressureException。尽管 RxJava 2.x Subject 考虑了 backpressure，但 2.x 的 PublishSubject 仍然会在下游跟不上节奏时抛出 MissingBackpressureException，而 2.x 的 ReplaySubject 则做了高效的无尽缓冲（类似于 onBackpressureBuffer）。

解决方案比较复杂，将在第二篇中进行讲解。

## 每次连接都传入一个新的 Subject

要解决重用的问题，我们可以把传入的 Subject 对象换成一个提供 Subject 对象的函数，每次 connect 之前都用它获取一个新的 Subject。

但这也带来了一个新的问题。因为在我们设置 OnSubscribe（调用 super）的时候，还没有 Subject 对象，所以我们需要记住还没有连接时想要订阅的 Subscriber，然后在连接时把它们订阅到创建的 Subject 上。

首先，我们需要管理更加复杂的状态了，我将创建一个 Connection 类来表示这些状态：

~~~ java
static final class Connection<T> {
    Subject<T, T> subject;                           // (1)
    List<Subscriber<? super T>> subscribers;         // (2)
    boolean connect;                                 // (3)
    final SerialSubscription parent;                 // (4)
     
    public Connection() {
        this.subscribers = new ArrayList<>();
        this.parent = new SerialSubscription();
    }
     
    public void setSubject(Subject<T, T> subject) {  // (5)
        // implement
         
    }
     
    public void subscribe(Subscriber<? super T> s) { // (6)
        // implement
    }
     
    public boolean tryConnect() {                    // (7)
        // implement
    }
}
~~~

先看看它的结构：

1. 我们需要保存 Subject 对象，所以订阅者可以随时订阅它；
2. 由于 connect 之前 Subject 并不存在，我们需呀把先来的 Subscriber 保存起来，并在 Subject 到来时把它们都订阅上去；
3. 每个 Connection 对象都只能连接一次（终止或者取消订阅之后，我们会创建一个新的 Connection 对象，后面会看到）；
4. 我们需要保存对上游的订阅。但不能直接使用 Subscription 成员，因为连接的过程可能较长，并发的连接可能会发现 Subscription 成员仍是 null（不像上面的简单实现，用了一个 atomic 引用来记录）。容器类则是始终不为 null 的，而且能保证恰当的取消订阅行为；
5. 一旦 Subject 到来，我们需要把它保存起来，同时把先来的 Subscriber 订阅上去；
6. 我们还需要在 ConnectableObservable 的构造函数中为 OnSubscribe 提供一个函数，根据当前连接的状态，妥善处理 Subscriber；
7. 最后，每个 Connection 对象只能连接一次，这一点在 tryConnect 中完成；

上面的（5~7）步虽然简单，但还是需要稍作解释：

~~~ java
public void setSubject(Subject<T, T> subject) {
    List<Subscriber<? super T>> list;
    synchronized (this) {
        this.subject = subject;
        list = subscribers;
        subscribers = null;
    }
    for (Subscriber<? super T> s : list) {
        subject.subscribe(s);
    }
}
~~~

在设置 Subject 时我们进行了同步，为了避免在设置 Subject 的同时有新的 Subscriber。这样，先来的 Subscriber 会在这个函数内订阅上去，后来的 Subscriber 则会在退出同步块后，直接订阅到 Subject 上，不必存进列表中。在同步块外订阅先来的 Subscriber，一方面可以避免死锁，另一方面也不会在循环中阻塞住并发的订阅。

接下来是 `subscribe()` 函数：

~~~ java
public void subscribe(Subscriber<? super T> s) {
    Subject<T, T> subject;
    synchronized (this) {
        subject = this.subject;
        if (subject == null) {
            subscribers.add(s);
            return;
        }
    }
    subject.subscribe(s);
}
~~~

在同步代码块中，我们检查 Subject 是否为 null，如果是（connect 还没有调用过），则把 Subscriber 加到列表中，否则直接订阅到 Subject 上。这里可以做一个优化，把 Subject 设置为 volatile 类型，然后利用 `double-check lock`，因为 Subject 只会被设置一次。

最后的 tryConnect() 就比较简单了：

~~~ java
public boolean tryConnect() {
   synchronized (this) {
        if (!connect) {
            connect = true;
            return true;
        }
        return false;
    }
}
~~~

在同步代码块中，如果当前 `connect` 是 false，那我们就把它置为 true，并返回 true，否则返回 false。返回 true 会触发连接的逻辑，返回 false 将会在 `connect()` 中返回 SerialSubscription（请见稍后的代码）。

新的类型我称之为 MulticastSupplier，结构如下：

~~~ java
public final class MulticastSupplier<T> 
extends ConnectableObservable<T> {
    public static <T> MulticastSupplier<T> create(        // (1)
            Observable<T> source, 
            Supplier<Subject<T, T>> subjectSupplier) {
        AtomicReference<Connection<T>> conn = 
            new AtomicReference<>(new Connection<>());    // (2)
         
        return new MulticastSupplier<>(
            source, subjectSupplier, conn);
    }
     
 
     
    final Observable<T> source;
    final Supplier<Subject<T, T>> subjectSupplier;
    final AtomicReference<Connection<T>> connection;      // (3)
     
     
    protected MulticastSupplier(Observable<T> source, 
            Supplier<Subject<T, T>> subjectSupplier,
            AtomicReference<Connection<T>> connection) {
        super(s -> {
            Connection<T> conn = connection.get();        // (4)
            conn.subscribe(s);
        });
        this.source = source;
        this.subjectSupplier = subjectSupplier;
        this.connection = connection; 
    }
 
    void replaceConnection(Connection<T> conn) {          // (5)
        Connection<T> next = new Connection<>();
        connection.compareAndSet(conn, next);
    }
     
    @Override
    public void connect(Action1<? super Subscription> connection) {
        // implement
    }
}
~~~

下面是详细的解析：

1. 由于在构造函数中，调用 super 之前不能调用非 static 方法，我们又必须在构造 MulticastSupplier 之前准备好 Connection 对象，这样 MulticastSupplier 和 OnSubscribe 才能使用它，所以我们用一个工厂方法做了这件事（_注意，也不是必须要有工厂方法，如果把 Connection 对象暴露出去，那就无需工厂方法了，但由于 Connection 是一个内部用的类，我们应该把它封装起来_）；
2. 由于 Connection 是可变的（我们会多次重新连接），所以我们需要一个原子引用类来容纳它，这让我们轻松保证原子性；此外在连接之前，我们需要记住先来的 Subscriber，所以这个原子引用初始值也不能是 null；
3. Connection 的原子引用在稍后的 connect() 函数中还会访问到；
4. OnSubscribe 被稍微修改了一下：我们获取当前的 Connection 对象，然后调用它的 subscribe() 方法；如果当时已经连接，那就会直接订阅到 Subject 上，否则会先存起来，连接时再订阅上去；
5. 最后，无论是断开连接，还是上游终止了，我们都会把 Connection 对象替换成新的；我们会先创建一个新的 Connection 对象，然后利用 CAS 把老的替换掉，这可以阻止老的 Subscription 断开新的连接；


最后我们看看 connect() 的实现：

~~~ java
@Override
public void connect(
        Action1<? super Subscription> connection) {
 
    Connection<T> conn = this.connection.get();          // (1)
     
    if (conn.tryConnect()) {                             // (2)
        Subject<T, T> subject = subjectSupplier.get();
         
        Subscriber<T> parent = new Subscriber<T>() {     // (3)
            @Override
            public void onNext(T t) {
                subject.onNext(t);
            }
             
            @Override
            public void onError(Throwable e) {           // (4)
                subject.onError(e);
                replaceConnection(conn);
            }
             
            @Override
            public void onCompleted() {
                subject.onCompleted();
                replaceConnection(conn);
            }
        };
         
        conn.parent.set(parent);                         // (5)
         
        parent.add(Subscriptions.create(() -> {          // (6)
            replaceConnection(conn);
        }));
         
        conn.setSubject(subject);                        // (7)
         
        connection.call(conn.parent);                    // (8)
         
        source.subscribe(parent);                        // (9)
    } else {
        connection.call(conn.parent);                    // (10)
    }
}
~~~

现在我们不需要 CAS 循环了，因为原子性的要求我们用其他方式保证了：

1. 首先我们获取当前的 Connection 对象；
2. 如果当前没有连接，我们就会标记为已连接（tryConnect 函数里面做的），然后执行连接逻辑，否则转到（10）；
3. 一旦我们有了 Subject 对象，我们就像之前那样，创建一个 Subscriber，把事件转发给 Subject；
4. 在收到终止事件时，我们希望尽早断开连接；这里有一个小小的决策，先 replaceConnection 还是先把终止事件发给 Subject？这取决于我们对竞争的容忍程度，现在的实现中，我们把竞争留给了下一个 Connection 而不是当前这个；
5. 然后我们把 parent 设置到 Connection 中，如果当前连接已经断开，这会立即取消订阅 parent；针对不同的连接-断开连接竞争的处理需求，我们可以在 `isUnsubscribed()` 返回 true 时直接退出，甚至都不用订阅到上游，而是直接返回一个已经取消订阅的 Subscription，我们也可以重试连接；后者就需要一个 CAS 循环了；
6. 当下游取消订阅时，我们要替换掉当前的 Connection；
7. 我们把 Subject 设置给当前的 Connection 对象，这有可能触发先到的 Subscriber 订阅到 Subject 中；
8. 在真的订阅上游之前，我们先把 `SerialSubscription`（注意不是 Subscriber）通过传入的 action 发布出去，使得可以同步取消订阅；
9. 然后我们把 parent 订阅到上游；
10. 如果 tryConnect 返回了 false，说明已经连接上了，那我们可以直接把当前 Connection 的 SerialSubscription 发布出去；这里需要注意，subscription 不能为 null；

## 总结

在本文中，我详细描述了 ConnectableObservable 的要求，并展示了两种简单的实现。

可能也有人会需要请求的协调功能，但这在 Multicast 和 MulticastSelector 中都没有实现。

考虑这一点也不是白费功夫，因为它在本系列的后半部分将会非常有用。

到目前为止，讲解的内容复杂度都在中等之下，因为因为数据流之间还没有进行交互，下一节，我们将在复杂度上更进一步，看看如何处理好下游的请求。

在我看来，这已经是大师级别的要求了，如果理解到位了，就打开了 RxJava 中最复杂的操作符实现的大门，保持跟进！
