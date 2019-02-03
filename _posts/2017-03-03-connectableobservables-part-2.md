---
layout: post
title: ConnectableObservables（二）
tags:
    - Operator
---

原文 [ConnectableObservables (part 2)](http://akarnokd.blogspot.com/2015/10/connectableobservables-part-2.html){:target="_blank"}。

## 介绍

在本系列上一篇文章中，我展示了怎样实现一个简单的 `ConnectableObservable`，它在连接之后，利用 `Subject` 把事件转发给所有的 `Subscriber`。

这一实现的主要问题是缺乏“请求管理”（request coordination，即 backpressure 支持），所有的数据流都运行在无尽缓冲模式下，开发者必须为每个 Subscriber 应用 `onBackpressureXXX` 操作符，而这要么会导致数据丢失（onBackpressureDrop），要么会导致缓冲区膨胀（onBackpressureBuffer）。

如果数据源 `Observable` 是 cold Observable，那我们就有可能让它只发出下游能够处理的数据量。为了实现这一功能，我们就需要实现**请求管理（request coordination）**。

## 请求管理

到目前为止，我们要实现的操作符需要每次处理单个 Subscriber 的请求。取决于操作符的具体逻辑，我们要么把请求传递给上游，要么把请求打包或者累积。

但如果考虑到多个 Subscriber，那问题的复杂度立马就提高到了新的维度，新的问题有哪些？

### 不同的请求数量

首先，不同的 Subscriber 可能请求不同的数量，有的请求很少，有的请求很多，有的甚至希望运行无尽模式（`request(Long.MAX_VALUE)`）。而且各个请求可能在任何时候到达，请求的数据量也可能是各不相同。

面临这样毫无章法的请求模式，我们应该向上游 Observable 发出怎样的请求？

我们有两种选择：

1. 发出所有 Subscriber 请求量的最小者；
2. 发出所有 Subscriber 请求量的最大者；

第一种选择让所有的 Subscriber 步调一致。这样做的好处就是我们无需做请求打包，或者数据缓冲处理。因为一旦上游发出了数据，所有的 Subscriber 都能消费所有数据。（如果请求频率很低，例如 1~10s 一次，我们倒是可以进行请求打包或者数据缓冲。）这样做的缺点则是整个链条都被最慢的 Subscriber 拖慢了，而一旦它“忘记”发出请求，那所有的 Subscriber 都收不到任何数据了。

第二种选择则让各个 Subscriber 有了更多自由发挥的空间，可以按照自己的步调运行。然而我们需要实现无尽的缓冲区（可能所有的 Subscriber 共享一个缓冲区，也可能每个 Subsciber 一个单独的缓冲区）。也就是说如果有一个 Subscriber 请求了 `Long.MAX_VALUE`，我们也必须向上游请求无限个，并且把所有的数据缓存起来。当然，考虑到操作符的具体逻辑，这一点可能并不是问题。

### Subscriber 可以来去自由

第二个问题是，Subscriber 的数量不是固定的，新的 Subscriber 会随时到来，老的 Subscriber 会随时离开。而这带来了一系列新的问题：

1. Subscriber 可能请求 `Long.MAX_VALUE`，但却在收到几个数据（甚至在收到数据之前）取消订阅；
2. Subscriber 可能不发出任何请求，（如果我们向上游请求最小数量）那其他的 Subscriber 都收不到数据；
3. Subscriber 可能在任何时候取消订阅，彼时我们需要“取消它的请求”；
4. 如果上游结束之前，所有的 Subscriber 都取消订阅了，那我们该怎么处理？

然而不幸的是，前两个问题是互斥的，前面已经提到了，我们要么保持步调一致，要么实现无尽缓冲。第三个问题则需要一个取消订阅的回调。

第四个问题取决于前两个问题的处理策略：

+ 在步调一致的方案中，我们有两种选择：要么用一个缓冲区把已经请求到的数据缓冲起来，等待新的 Subscriber；要么就要在新的 Subscriber 到来之前丢弃掉上游发过来的数据；
+ 在无尽缓冲模式中，我们可以继续缓冲，或者开始丢弃数据；

### RxJava 的做法

RxJava 有两个操作符返回 `ConnectableObservable`：`publish()` 和 `replay()`。在很长一段时间里，它们都完全忽略了 backpressure 这回事，就像前一篇文章中介绍的 `MulticastSupplier` 一样。

这两个操作符分别在 `1.0.13` 和 `1.0.14` 版本中被重写，以支持 backpressure，所以前面提到的问题就考虑了进来。它们的实现方案如下：

`publish()` 采用了步调一致方案，并使用了一个固定的预取缓冲区：这个缓冲区只有在所有的 Subscriber 都可以接收数据时才开始漏出数据（以及填充数据）。如果没有 Subscriber，那它就开始“缓慢地丢掉”数据，每次请求一个数据，并把它丢掉。

`replay()` 采用了无尽缓冲方案，因为不管是用有界还是无界的缓冲区，我们都需要重放上游的所有数据。大家可能会思考，如果重放的元素是受时间和缓冲区大小限制的，我们为什么还需要无限缓冲区？因为这些操作符和 Subject 一样，必须保证连续、无丢失地传递数据。如果有一个 Subscriber 请求了一个数据 A，然后 sleep，那它下次请求的时候，收到的数据必须是 A 的下一个数据，不管其他的 Subscriber 已经接收了多少数据。

### 断开连接的影响

RxJava 里实现的操作符中，有一个问题没有处理，在这里值得一提。如果我们取消订阅了 `connect()` 函数返回的 `Subscription`（断开连接），那上游就不会发出任何事件了。

这里的问题是断开连接可能导致下游的 Subscriber 夯住：它们不会收到任何事件了（除了那些已经缓冲起来了的事件）。在 Java 8 的 `CompletableFuture` 中我们也有类似的问题，我们如果取消了 `Future`，那还在等待结果的使用者该怎么办？

Java 8 中抛出了一个 `CancellationException`，这样等待的任务就都可以终止了。但这不是 RxJava 的做法（1.x 和 2.x 都一样），RxJava 目前就是让 Subscriber 夯住。

这个问题也可能不局限于 ConnectableObservable，有一段时间里，RxAndroid 0.x 包含了一个操作符，它会在 Activity/Fragment 的 onDestroy 时取消所有应用过的数据流，这就会导致 Subscriber 收不到终止事件了。我建议在这种情况下发出一个 `onError` 或者 `onCompleted` 事件。但这个问题并没有解决，而是在 RxAndroid 1.0 之前把这个操作符在删掉了。

_我额外提一句，我不记得社区里有人提到过这个问题，似乎没人被这个问题影响到。这和其他很多模糊、边界的情况一样，如果我没有提到它们，似乎都没人发现。_

### 终止事件的影响

上游的 Observable 可能会正常停止，这时 ConnectableObservable 把这个终止事件转发给 Subscriber。

这时如果新来了一个 Subscriber，我们怎么处理？终止事件是否意味着断开连接？这个 Subscriber 是否应该立即收到终止事件，就像 `PublishSubject` 一样？

同样，这个问题的处理方案也取决于业务逻辑。RxJava 的方案是把终止事件等同于断开连接，后来的 Subscriber 不会收到终止事件，但是会被保存起来，在下次调用 connect 时会再被订阅。

这样做的好处是，开发者可以在上游 Observable 运行起来之前先准备 Subscriber，这样可以避免丢失事件。坏处就是我们一定要记得再次调用 connect，否则 Subscriber 就不会收到任何事件。

## 收集者和发射者家族

在开始看代码之前，我想先提一下处理多个数据源或者多个订阅者的操作符的基本模式。

我编写了很多这样的操作符，我注意到它们都用了同样的模块和方法：

1. 它们都需要记录 Subscriber，无论是下游的 Subscriber 还是订阅到上游的 Subscriber；而这一记录功能都用了[基于数组的 copy-on-write 方式实现的容器类](/AdvancedRxJava/2016/07/29/operator-concurrency-primitives-subscription-containers-3/index.html)；
2. 它们都用了发射者循环（基于 `synchronized`）或者队列漏（基于原子类），而这两者都需要在很多情形中触发：上游发出事件时，新的下游到来时，下游发来请求时，或者下游取消订阅时；
3. 它们的循环体里都需要一些预处理：搞清楚当前 Subscriber 的情况，选择一个上游开始漏出，或者按照某种方式把上游的数据结合起来；
4. 最终，事件被发送到了 Subscriber 那里，并且向上游 Observable 发出请求补充数据；

## 实现哪一个操作符？

既然我们已经明确了问题，那就开始实现一个 ConnectableObservable 来解决请求管理的问题吧。

我考虑了一下应该实现哪一个操作符。我首先想到的是展示一下怎么实现和 `AsyncSubject` 或者 `BehaviorSubject` 对应的操作符（就像 `publish()` 之于 `PublishSubject`），但是前者可以通过 `replay()` 很轻易地实现：

~~~ java
public ConnectableObservable<T> async() {
    return takeLast(1).replay();
}
~~~

实现 BehaviorSubject 对应的操作符则稍微麻烦一点，一种很简单的实现可以是这样：

~~~ java
public ConnectableObservable<T> behave() {
    return replay(1);
}
~~~

但是这一实现不具备 BehaviorSubject 终止后的特性：后来的 Subscriber 只会收到一个终止事件，但用 replay 实现时，后来的 Subscriber 会收到最后一个事件以及终止事件。

为了减小烧脑程度，即便是最简单的 `publish()` 操作符，我也决定不去实现它的各种版本了。

## Publish (or die)

首先我们列出一下我们想要达到的所有效果：

1. 我们需要实现步调一致的请求管理，并实现预取（以提升效率）；
2. 断开连接的行为应该是可以配置的：不发出任何事件，发出 error，或者发出 completed；
3. 上游终止后，新的 Subscriber 应该被保存，等待新的 connect 操作；
4. 我们允许 error 立即终止事件流（读者可以自行实现延迟 error）；
5. 预取缓冲区的大小将是 2 的幂次；

明确了要求之后，我们先看看整体类结构：

~~~ java
public class PublishConnectableObservable<T> 
extends ConnectableObservable<T> {
 
    public enum DisconnectStrategy {                           // (1)
        NO_EVENT,
        SEND_ERROR,
        SEND_COMPLETED
    }
     
    public static <T> PublishConnectableObservable<T> 
    createWith(                                               // (2)
            Observable<T> source, 
            DisconnectStrategy strategy) {
        State<T> state = new State<>(strategy, source);
        return new PublishConnectableObservable<>(state);
    }
     
    final State<T> state;
     
    protected PublishConnectableObservable(State<T> state) {  // (3)
        super(state);
        this.state = state;
    }
     
    @Override
    public void connect(
            Action1<? super Subscription> connection) {       // (4)
        state.connect(connection);
    }
}
~~~

目前还没什么特殊之处：

1. 我们用一个 enum 来标记断连的策略；
2. 我们需要一个工厂方法（前文已经多次提到了这一点，我们需要在父类构造函数之前 new 一个对象，并且赋值给一个变量，并且在后面让 OnSubscribe 引用，要么就把 State 类公开给使用者，要么就用工厂方法封装起来）；
3. 我们让 State 实现 OnSubscribe 接口，节省一次内存分配；
4. 最后我们把 connect 调用委托给 State 对象，这样我们的代码更简洁一些；

接下来是和上一篇文章中比较类似的 State 类结构：

~~~ java
static final class State<T> implements OnSubscribe<T> {
    final DisconnectStrategy strategy;
    final Observable<T> source;
     
    final AtomicReference<Connection<T>> connection;      // (1)
       
    public State(DisconnectStrategy strategy, 
            Observable<T> source) {                       // (2)
        this.strategy = strategy;
        this.source = source;
        this.connection = new AtomicReference<>(
            new Connection<>(this)
        );
    }
         
    @Override
    public void call(Subscriber<? super T> s) {           // (3)
        // implement
    }
         
    public void connect(
        Action1<? super Subscription> disconnect) {       // (4)
        // implement
    }
         
    public void replaceConnection(Connection<T> conn) {   // (5)
        Connection<T> next = new Connection<>(this);
        connection.compareAndSet(conn, next);
    }
}
~~~

State 类将要负起连接、订阅、重新连接的责任：

1. 由于我们需要重新连接，我们把当前的 Connection 对象保存在 AtomicReference 中；
2. 初始化成员变量，当前 Connection 被初始化为“未连接”；
3. `call()` 函数负责处理订阅；
4. connect 函数负责处理连接；
5. 最后，如果上游终止了，或者外部主动断开了连接，我们需要用一个全新的 Connection 对象替换老的，我们用一个原子操作，避免因为竞争而把其他线程设置的新 Connection 对象替换掉；

在开始复杂的实现之前，我们先来看看两个很简单的类。首先是订阅到上游 Observable 的 Subscriber 类：

~~~ java
static final class SourceSubscriber<T> 
extends Subscriber<T> {
    final Connection<T> connection;
    public SourceSubscriber(
            Connection<T> connection) {    // (1)
        this.connection = connection;
    }
    @Override
    public void onStart() {
        request(RxRingBuffer.SIZE);        // (2)
    }
 
    @Override
    public void onNext(T t) {
        connection.onNext(t);              // (3)
    }
 
    @Override
    public void onError(Throwable e) {
        connection.onError(e);
    }
 
    @Override
    public void onCompleted() {
        connection.onCompleted();
    }
     
    public void requestMore(long n) {      // (4)
        request(n);
    }
}
~~~

这个类同样也都是委托调用：

1. 保存后面我们将要委托的 Connection 对象；
2. 如果我们订阅到了源 Observable 上，我们发出一个小量的请求（读者可以把这个请求量变成一个参数）；
3. 然后我们把上游的事件都委托到 Connection 对象上，而 Connection 类实现了 `Observer`，这样就很方便了；
4. 我们需要补充被消耗的数据，但 request 是 protected 的，所以我们用 requestMore 把它暴露出去；

接下来是 Producer 和 Subscription 的实现，用来负责取消订阅以及请求处理：

~~~ java
static final class PublishProducer<T> 
implements Producer, Subscription {
    final Subscriber<? super T> actual;
    final AtomicLong requested;
    final AtomicBoolean once;
    volatile Connection<T> connection;             // (1)
     
    public PublishProducer(
            Subscriber<? super T> actual) {
        this.actual = actual;
        this.requested = new AtomicLong();
        this.once = new AtomicBoolean();
    }
     
    @Override
    public void request(long n) {
        if (n < 0) {
            throw new IllegalArgumentException();
        }
        if (n > 0) {
            BackpressureUtils
                .getAndAddRequest(requested, n);
            Connection<T> conn = connection;       // (2)
            if (conn != null) {
                conn.drain();
            }
        }
    }
     
    @Override
    public boolean isUnsubscribed() {
        return once.get();
    }
     
    @Override
    public void unsubscribe() {
        if (once.compareAndSet(false, true)) {
            Connection<T> conn = connection;       // (3)
            if (conn != null) {
                conn.remove(this);
                conn.drain();
            }
        }
    }
}
~~~

现在变得有趣一点了：

1. 出于以下两个原因，我们必须保存当前正在处理的 Connection 对象：我们需要通知 Connection 对象有 Subscriber 可以接收新的数据了（发出了请求）；如果有 Subscriber 取消订阅了，我们也需要通知 Connection 对象可以接收新的数据了（离开的那个可能是拖后腿的）；
2. 由于 request() 是异步的，可能此时 Connection 对象还没有设置，那我们就需要在之后合适的时机调用 drain() 方法（下文讲解）；
3. 同样，unsubscribe() 也是异步的，我们也需要确保 Connection 已经设置，然后再把自己从 Subscriber 数组中移除（下文讲解）；注意幂等性由 `once` 变量来保证；

最后，是 Connection 的结构：

~~~ java
@SuppressWarnings({ "unchecked", "rawtypes" })
static final class Connection<T>
 implements Observer<T> {                             // (1)
 
    final AtomicReference<PublishProducer<T>[]>
        subscribers;
    final State<T> state;
    final AtomicBoolean connected;
     
    final Queue<T> queue;
    final AtomicReference<Throwable> error;
    volatile boolean done;
 
    volatile boolean disconnected;
     
    final AtomicInteger wip;
     
    final SourceSubscriber parent;
     
     
    static final PublishProducer[] EMPTY = 
        new PublishProducer[0];
 
    static final PublishProducer[] TERMINATED = 
        new PublishProducer[0];
     
    public Connection(State<T> state) {               // (2)
        this.state = state;
        this.subscribers = new AtomicReference<>(EMPTY);
        this.connected = new AtomicBoolean();
        this.queue = new SpscArrayQueue(
            RxRingBuffer.SIZE);
        this.error = new AtomicReference<>();
        this.wip = new AtomicInteger();
        this.parent = createParent();
    }
     
    SourceSubscriber createParent() {                 // (3)
        // implement
    }
     
    boolean add(PublishProducer<T> producer) {        // (4)
        // implement
    }
     
    void remove(PublishProducer<T> producer) {
        // implement
    }
     
    void onConnect(
         Action1<? super Subscription> disconnect) {  // (5)
        // implement
    }
     
    @Override
    public void onNext(T t) {                         // (6)
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
     
    void drain() {                                    // (7)
        // implement
    }
     
    boolean checkTerminated(boolean d, 
        boolean empty) {
        // implement
    }
}
~~~

方法和变量的名字现在看起来应该很眼熟了：

1. Connection 类需要管理众多状态：当前的 Subscriber 数组；事件队列以及终止事件的容器；已经连接和断开连接的标识；队列漏需要的工作计数；订阅到上游 Observable 的 Subscriber（parent）；以及标记空状态和终止状态的 Subscriber 数组；
2. 在构造函数中我们初始化各个状态；
3. 创建 SourceSubscriber 时我们还需要一些准备工作，所以我把它单独抽离为一个函数；
4. 维护 Subscriber 的 copy-on-write 处理是通过 add 和 remove 完成的，就像在[处理 Subject 时](/AdvancedRxJava/2016/10/05/subjects-part-3/index.html)那样，而且用了[基于数组的 Subscription 容器类](/AdvancedRxJava/2016/07/29/operator-concurrency-primitives-subscription-containers-3/index.html)；
5. 上游事件在 onXXX 中进行处理；
6. 最后，队列漏的实现还需要 drain 和 checkTerminated 函数；

## 烧脑的部分来啦！（The meltdown）

到目前为止上面的类结构和已经实现了的函数都没什么特殊之处，但真正复杂的内容现在开始了。我将逐个实现上面没有实现的函数，并讲解并发方面的考虑。

我建议大家先休息一会儿，补充点能量，放松下心情。

（_译者注：我在翻译完上一节，看到这句话之前机智的休息了一段时间。_）

准备好了吗？让我们开始吧 :)

### `State.call`

这个函数负责处理新来的 Subscriber，它需要考虑到上游可能已经终止了，或者连接已经断开：

~~~ java
@Override
public void call(Subscriber<? super T> s) {
    PublishProducer<T> pp 
        = new PublishProducer<>(s);
     
    s.add(pp);
    s.setProducer(pp);                                // (1)
 
    for (;;) {
        Connection<T> curr = connection.get();
         
        pp.connection = curr;                         // (2)
        if (curr.add(pp)) {                           // (3)
            if (pp.isUnsubscribed()) {                // (4)
                curr.remove(pp);
            } else {
                curr.drain();                         // (5)
            }
            break;
        }
    }
}
~~~

1. 我们首先创建一个 PublishProducer，并把它设置给 Subscriber，以处理请求和取消订阅；
2. 接下来我们拿到当前的 Connection 对象，并把它设置给 PublishProducer，这样在它内部就可以调用 drain() 函数了；
3. 我们尝试把 PublishProducer 加入到 Connection 维护的数组中去，如果失败了，则说明当前的连接已经终止了（上游终止，或者下游断开了连接），那我们就继续循环尝试（_连接终止时，Connection 对象会被替换，所以这个循环就像 CAS 一样，不会执行太多次_）；
4. 即便 add 成功，下游可能已经在 add 前取消订阅，那在 PublishProducer 的 unsubscribe 函数中，我们就无法把自己从 Connection 中移除（因为还没有 add 进去）；那么在这里再次检查并处理这一点，我们就能保证已经取消订阅的 Subscriber 不会继续保存在 Connection 对象的内部数组中；
5. 一旦 add 成功且未取消订阅，那我们就需要调用 drain 函数，因为在对 PublishProducer 的并发调用时，可能其 Connection 尚未设置，所以这里我们要把这些调用通知给 Connection；

### `State.connect`

这个函数负责单次连接一个尚未连接的 Connection 对象，以及（通过回调）返回 Subscription 对象，以便断开连接。

~~~ java
public void connect(Action1<? super Subscription> disconnect) {
    for (;;) {
        Connection<T> curr = this.connection.get();
         
        if (!curr.connected.get() && 
                curr.connected.compareAndSet(false, true)) {  // (1)
            curr.doConnect(disconnect);
            return;
        }
        if (!curr.parent.isUnsubscribed()) {                  // (2)
            disconnect.call(curr.parent);
            return;
        }
         
        replaceConnection(curr);                              // (3)
    }
}
~~~

这个函数也可能和终止事件以及断开连接发生竞争，所以我们必须在建立新的连接时考虑这些情况：

1. 我们首先拿到当前的 Connection 对象，如果此时还没有连接，那我们就尝试建立连接，如果连接成功，那我们就调用 doConnect 函数，在其中执行订阅的逻辑；
2. 否则，我们就检查当前的连接是否已经取消。如果没有取消，那我们就可以把它（通过回调）返回给调用方了。_注意，这里有一个小小的竞争窗口，我们的 if 判断通过了，但在调用 call 之前连接却被断开了。要解决这一问题，我们要么在断开连接和建立连接时使用阻塞的同步方式，要么就要实现串行访问。但在实际使用中，这个竞争基本不会发生，所以可以忽略_；
3. 最后，如果当前的连接已经断开，那我们就把当前的 Connection 对象替换成一个全新的、尚未连接的 Connection 对象，然后继续这一循环；

### `Connection.createParent`

这个函数创建一个 SourceSubscriber 对象，并根据断连策略对它进行设置：

~~~ java
SourceSubscriber createParent() {
    SourceSubscriber parent = new SourceSubscriber<>(this);
     
    parent.add(Subscriptions.create(() -> {
        switch (state.strategy) {
        case SEND_COMPLETED:
            onCompleted();
            break;
        case SEND_ERROR:
            onError(new CancellationException("Disconnected"));
            break;
        default:
            disconnected = true;
            drain();
        }
    }));
     
    return parent;
}
~~~

这个函数会创建一个 SourceSubscriber 对象，并向其中加入一个 Subscription 以便在连接被断开时执行相关逻辑。根据断连策略，我们会发出一个 `onCompleted`，或者用 CancellationException 发出一个 `onError`，或者把 disconnected 置为 true 之后开始 drain 函数（onXXX 函数中也会调用 drain，所以前两种策略这里我们无需调用 drain）。

我们需要 disconnected 这个标识，因为我们不能用 isUnsubscribed：如果是上游正常终止了，那 isUnsubscribed 会返回 false，那我们就一定不会向下游发出事件了（始终都是 `NO_EVENT` 的行为了）。

### `Connection.add` 和 `Connection.remove`

基于数组的 copy-on-write 算法现在对我们来说应该很熟悉了（_不熟悉也没关系，逻辑很直观_），为了完整性，这里还是给出代码：

~~~ java
boolean add(PublishProducer<T> producer) {
    for (;;) {
        PublishProducer<T>[] curr = subscribers.get();
        if (curr == TERMINATED) {
            return false;
        }
         
        int n = curr.length;
         
        PublishProducer<T>[] next = new PublishProducer[n + 1];
        System.arraycopy(curr, 0, next, 0, n);
        next[n] = producer;
        if (subscribers.compareAndSet(curr, next)) {
            return true;
        }
    }
}
 
void remove(PublishProducer<T> producer) {
    for (;;) {
        PublishProducer<T>[] curr = subscribers.get();
        if (curr == TERMINATED || curr == EMPTY) {
            return;
        }
         
        int n = curr.length;
         
        int j = -1;
        for (int i = 0; i < n; i++) {
            if (curr[i] == producer) {
                j = i;
                break;
            }
        }
         
        if (j < 0) {
            break;
        }
        PublishProducer<T>[] next;
        if (n == 1) {
            next = EMPTY;
        } else {
            next = new PublishProducer[n - 1];
            System.arraycopy(curr, 0, next, 0, j);
            System.arraycopy(curr, j + 1, next, j, n - j - 1);
        }
        if (subscribers.compareAndSet(curr, next)) {
            return;
        }
    }
}
~~~

### `Connection.onXXX`

4 个 onXXX 函数比较类似，所以这里一起讲解：

~~~ java
void onConnect(
         Action1<? super Subscription> disconnect) {        // (1)
    disconnect.call(this.parent);
       
    state.source.unsafeSubscribe(parent);
}
     
@Override
public void onNext(T t) {                                   // (2)
    if (queue.offer(t)) {
        drain();
    } else {
        onError(new MissingBackpressureException());
        parent.unsubscribe();
    }
}
 
@Override
public void onError(Throwable e) {
    if (!error.compareAndSet(null, e)) {                    // (3)
        e.printStackTrace();
    } else {
        done = true;
        drain();
    }
}
 
@Override
public void onCompleted() {                                 // (4)
    done = true;
    drain();
}
~~~

让我们瞧一瞧：

1. 之所以要把这个 Action1 一路带到这里，而不是直接在 State.connect 的（2）处就调用回调，是因为我们必须在订阅到源 Observable 之前执行回调，以便于外部的同步取消订阅；
2. onNext 里我们尝试把数据加入到队列中，如果成功我们就尝试 drain。如果队列满了，我们就用 onError 发出一个 `MissingBackpressureException` 并且取消订阅，这说明上游并没有处理好 backpressure，或者压根没实现；
3. 由于我们可能在多处收到 onError（上游、断开连接、onNext 中），我们要保证只有一个错误被发往下游，所以我们用一个 AtomicReference 来记录这个错误。在这里，第一个错误将被发往下游，其他的只会在控制台打印日志。如果 CAS 成功，我们就设置 done 标记，然后调用 drain 函数做事；
4. onCompleted 确实可能会被调用多次，但由于这里只是把设置 done 标记，所以无需 CAS。此外，由于断连策略，onError 和 onCompleted 也确实可能发生竞争，但由于它们的区别仅仅是 error 的容器里面是否会放一个异常，所以也不会导致实际的问题。另外，由于我们在 onConnect 中用了 unsafeSubscribe，所以我们也不能在 onCompleted 中调用 SourceSubscriber.unsubscribe，如果上游正常终止，且下游的断连策略是 `SEND_ERROR`，调用 unsubscribe 就会导致下游收到错误；

### `Connection.drain`

这个函数无疑是整个操作符最核心的部分，而且由于它使用的各个变量都可能被并发修改，处理这一问题的逻辑也导致这个函数是整个操作符最复杂的部分。接下来我将拆分为多个步骤讲解：

首先，它需要普通队列漏所使用的 wip 和 missed 计数器：

~~~ java
void drain() {
    if (wip.getAndIncrement() != 0) {
        return;
    }
     
    int missed = 1;
     
    for (;;) {
 
        if (checkTerminated(done, queue.isEmpty())) {
            return;
        }
 
        // implement rest
        
        missed = wip.addAndGet(-missed);
        if (missed == 0) {
            break;
        }
    }
}
~~~

这还没啥花哨的东西，wip 有两个作用，一是充当串行访问的 0-1 计数器，二是大于 1 时充当 missed 计数器。

在循环中，第一件事就是通过 checkTerminated 检查终止状态（下文讲解）。它将检查终止事件和断连状态，并作出响应。这一步在请求管理之前，因为终止事件并不属于 backpressure 关心的内容，它可以在下游发出任何请求之前发生。

下一步就是进行请求管理。由于我们的策略是步调一致，所以我们必须询问所有的 Subscriber 它们的请求量，然后把最小值发给上游，注意这个最小值可能是 0。

~~~ java
//... checkTerminated call
 
PublishProducer<T>[] a = subscribers.get();
 
int n = a.length;
long minRequested = Long.MAX_VALUE;
 
for (PublishProducer<T> pp : a) {
    if (!pp.isUnsubscribed()) {
        minRequested = Math.min(minRequested, pp.requested.get());
    }
}
 
// ... missed decrementing
~~~

此时，n 可能是 0。如果还没有 Subscriber，我们就进入“缓慢丢弃”模式：

~~~ java
// ... minRequested calculation
 
if (n == 0) {
    if (queue.poll() != null) {
        parent.requestMore(1);
    }
} else {
    // implement rest           
}
 
// ... missed decrementing
~~~

我们从队列中取出一个元素并丢弃掉（不使用），然后再请求一个新的数据。注意这里的“缓慢”取决于上游的速度，如果我们的需求是没有 Subscriber 时什么也不做，我们可以把 if 语句简化为 `if (n != 0) { }`，但不能省略这个检查（只需要让下面的分支确保 n 不为 0 即可）。

如果我们知道当前已经有了 Subscriber，并且计算出了最小请求量，我们就可以尝试从队列中漏出数据并发往下游了：

~~~ java
    // if n != 0 branch
 
    if (checkTerminated(done, queue.isEmpty())) {   // (1)
        return;
    }
 
    long e = 0L;
    while (minRequested != 0) {
 
        boolean d = done;
        T v = queue.poll();
         
        if (checkTerminated(d, v == null)) {        // (2)
            return;
        }
         
        if (v == null) {
            break;
        }
 
        // final detail to implement
         
        minRequested--;                             // (3)
        e++;
    }
     
    if (e != 0L) {                                  // (4)
        parent.requestMore(e);
    }
 
// end of n != branch
~~~

这部分代码看起来也应该比较熟悉了，我们再次检查终止状态（1），当然这是比较激进的策略，是可选的。接下来我们在循环中从队列中取出数据，直到 minRequested 为 0 或者队列为空。在循环里面我们也检查终止状态（2），以及发射计数（3）。退出循环后，如果我们确实发出了数据，我们就向 SourceSubscriber 请求补充数据（4）。

最后一部分就是把数据发往每个下游了：

~~~ java
// ... v == null check
 
for (PublishProducer<T> pp : a) {
    pp.actual.onNext(v);
    if (pp.requested.get() != Long.MAX_VALUE) {
        pp.requested.decrementAndGet();
    }
}
 
// ... minRequested--
~~~

对每一个 PublishProducer（也就是下游的 Subscriber），我们把数据发送给它，如果它的请求量不是 `Long.MAX_VALUE`（处于有限模式下），我们就递减它的请求计数。

也没那么可怕，对吧？

### `Connection.checkTerminated`

checkTerminated 要做的事情比以前的版本更多，因为它要把终止事件发送给所有的下游 Subscriber，同时还要保证终止之后，新 Subscriber 的 add 操作不会成功。

~~~ java
boolean checkTerminated(boolean done, boolean empty) {    // (1)
    if (disconnected) {                                   // (2)
        subscribers.set(TERMINATED);
        queue.clear();
        return true;
    }
    if (done) {
        Throwable e = error.get();
        if (e != null) {
            state.replaceConnection(this);                // (3)
            queue.clear();
 
            PublishProducer<T>[] a = 
                subscribers.getAndSet(TERMINATED);        // (4)
             
            for (PublishProducer<T> pp : a) {             // (5)
                if (!pp.isUnsubscribed()) {
                    pp.actual.onError(e);
                }
            }
             
             
            return true;
        } else if (empty) {
            state.replaceConnection(this);                // (6)
 
            PublishProducer<T>[] a = 
                subscribers.getAndSet(TERMINATED);
             
            for (PublishProducer<T> pp : a) {
                if (!pp.isUnsubscribed()) {
                    pp.actual.onCompleted();
                }
            }
             
            return true;
        }
    }
    return false;
}
~~~

它的工作机制如下：

1. 这个函数只接收 done 和 empty 标记变量，无需某个单独的 Subscriber 或者 Subscriber 数组；
2. 由于 disconnected 只会在断开连接且断连策略是 `NO_EVENT` 时才会被设置，所以这是我们只需要把 Subscriber 数组置为 TERMINATED 即可。如果此时仍有 Subscriber 未取消订阅，那它就不会收到任何事件了；
3. 如果 done 为 true，而且有错误发生，那我们先把当前的 Connection 对象替换掉，以免有新的 Subscriber 订阅到已经终止的 Connection 上来；
4. 清空了队列的数据之后，我们把 Subscriber 数组置为 TERMINATED；
5. 这样所有的 Subscriber 都会收到终止事件，而且漏循环也可以退出了；
6. 当上游终止，且队列中的数据已经漏空的时候，我们也执行同样的逻辑；

## 测试一下

我们终于完成了 RxJava 史上最复杂的一个操作符，现在让我们用一个简单的单元测试来犒劳一下自己，看看 backpressure 以及断连策略是否正常工作：

~~~ java
Observable<Integer> source = Observable.range(1, 10);

TestSubscriber<Integer> ts = TestSubscriber.create(5);

PublishConnectableObservable<Integer> o = createWith(
    source, DisconnectStrategy.SEND_ERROR);

o.subscribe(ts);

Subscription s = o.connect();

s.unsubscribe();

System.out.println(ts.getOnNextEvents());
ts.assertValues(1, 2, 3, 4, 5);
ts.assertNotCompleted();
ts.assertError(CancellationException.class);
~~~

测例应该打印出 `[1, 2, 3, 4, 5]`，并且没有错误。

## 总结

在这一片又长又烧脑的文章中，我介绍了 ConnectableObservable 处理下游的请求时，要满足的要求，以及面临的问题。紧接着我实现了一个类似于 `publish()` 的 ConnectableObservable，它能配置断连策略，以避免导致下游夯住。

然而，publish 并不是 RxJava 中最复杂的操作符。它还不是 replay，尽管有界的缓冲区看起来比 PublishConnectableObservable 更复杂一点，但这部分复杂度都在缓冲区的边界管理上。而且这个操作符也不是最常用的操作符，这也让它更简单一点，因为状态冲突更少。不过最复杂的操作符因为请求管理太过复杂，连我也不确定能否实现有界缓冲区的版本。

但这也足够揭开这部分内容的神秘面纱了！在本系列下一篇中，我将详细讲解怎样实现一个类似于 `replay()` 的 ConnectableObservable。
