---
layout: post
title: FlatMap 技术揭秘（一）
tags:
    - Completable
---

原文 [FlatMap (part 1)](http://akarnokd.blogspot.com/2016/02/flatmap-part-1.html){:target="_blank"}。

## 介绍

在本文中，我将开始对使用最广泛、误解也最多同时也是最复杂的操作符 flatMap 进行一次里里外外的大揭秘。

**flatMap** 是迄今为止最有用的操作符，因为它能让我们把简单的输入事件转换为任何我们想要的事件，我们可以控制时间、位置甚至事件数量。**flatMap** 被误解是因为诞生太晚，没有足够的时间充分展示它的使用，而且使用它经常伴随着函数式编程的技术挑战。最后，它的复杂之处在于它需要处理好单一消费者的 backpressure，以及从多个数据源请求数据，而且我们不知道哪个数据源会发出数据，有可能都会发出数据。

**flatMap** 有一个兄弟操作符：`merge`。merge 让我们可以把多个事件流合并为一个数据流，同时保证 Observer 的要求：onXXX 必须逐个发生，不能并发到来，而且还要符合 `onNext* (onError|onCompleted)?` 的模式。这一点很重要，因为即便每个数据源都保证了上述要求，但同时订阅多个数据源，就有可能出现违背的情况了。当然，flatMap 也有同样的要求，那为什么需要有两个不同的操作符呢？

答案是为了方便，以及考虑到不同的使用模式。**flatMap** 是一个“序列内”的操作符，它把上游的单个事件转换为一个 Observable，这一转换在一个回调函数中实现，在回调中它又会订阅到由它产生的 Observable 上，进行相关的处理。而 **merge** 则作用于一个二维的序列：发出 Observable 的 Observable。merge 倒是不涉及回调函数，但它会订阅到外层 Observable 发出的所有内层 Observable。

有趣的是，它俩可以互相表述：

~~~ java
Func1<T, Observable<R>> f = ...
source.flatMap(f) == Observable.merge(source.map(f))
 
 
Observable<Observable<R>> o = ...
Observable.merge(o) == o.flatMap(v -> v)
~~~

第一种情况中，我们可以通过 map 函数把源 Observable 中的数据转换为 `Observable<R>`，然后又把它们通过 merge 合并。如果大家去看看 RxJava 1.x 的源码，就会发现 flatMap 正是这样实现的。在第二种情况中，给定一个二维的数据源，当我们把内层的每个 `Observable<R>` 当做一个数据 v，那它就已经是我们可以返回的形式了，_直接返回就起到了 merge 的效果_。

我们可以把 flatMap 看做是 fork-join 操作的 join 部分，所有的线程又聚到一起，形成了一个单一的数据流。然而没有任何机制保证这一汇聚时刻何时到来。

举个例子，假如我们有一系列的产品 ID，我们需要通过网络请求获取一些额外的信息，利用 Retrofit 可以很方便的实现 RxJava 的支持。我们知道网络访问以及数据库访问的响应时间都是不确定的，后发出的网络请求可能先返回。我们获得的数据只能说是具有某种顺序的，有时顺序并不重要，但有时顺序很重要。

无论我们强调了多少次 flatMap 的这种不定序性，与之相关的问题总是被大家提出来，为什么会这样？

问题在于我们向读者介绍 flatMap 的方式：我们展示的是一个完全同步的场景，把一个序列中的每个元素转换为一个子序列。

~~~ java
Observable.range(1, 10).flatMap(v -> Observable.range(v, 2))
.subscribe(System.out::println);
~~~

这是一个完全同步的序列，输出结果也完全有序。而 concatMap 用的也是这个例子，但 concatMap 的确会保证顺序，这个例子中它俩的输出是一样的。那么让我们构造一个异步的例子，其中我们可以轻易地看到输出的顺序是会被打乱的：

~~~ java
Observable.range(1, 10)
.flatMap(v -> Observable.just(v).delay(11 - v, TimeUnit.SECONDS))
.toBlocking()
.subscribe(System.out::println);
~~~

在这个例子中，我们把每个数据转换为只有一个数据的 Observable，但它会经过一个 delay 操作，原数据越大，延迟越小。最终，我们将看到完全逆序的输出。

**flatMap** 也在异步延续的场景中扮演着重要的角色。也就是说当一个异步的计算/网络任务完成之后，我们希望根据先前任务的**唯一一个**输出数据来开始另一个异步任务。这里我强调了唯一性。因为据我所知，还没有支持 RxJava 的网络库会在一次请求中发出多个数据，而是只会发出一个数据（这个数据可能是一个 `List`，但它也只是一个对象）。所以当我们使用 flatMap 时，我们也很少用到它同时启用多个 Observable 的特性。

使用 **flatMap** 的最后一个特性就是它能改变发往下游的数据数量。实现不同的回调函数（map 函数），我们可以让上游的每个数据变成没有输出、刚好一个输出、多个输出甚至是发出一个错误。我们只需要在回调中返回 `empty()`，`just()`，一个 Observable 调用链或者是 `error()`。

经常有人问怎么在普通的 map 操作中抛出错误，其实我们只需要抛出异常而不是返回结果就可以了。如果我们要抛出的异常是 RuntimeException，那就可以直接抛出，RxJava 会在 map 操作符内部把它转换为一次 onError 事件。但如果我们要抛出的是一个 checked Exception（例如 IOException），那我们就不能这么干了。要么我们得把它包装为一个 RuntimeException，并在处理的地方解包装，要么我们就得实现一个自定义的 map 操作符，并在其中处理好 checked Exception。

另一种方案就是使用 flatMap，因为我们可以返回 `Observable.error()`，这样我们就能直接把异常转变为 onError 事件，而无需包装和解包装了：

~~~ java
Observable.range(1, 10)
.flatMap(v -> {
    if (v < 5) {
        return Observable.just(v * v);
    }
    return Observable.<Integer>error(new IOException("Why not?!"));
})
.subscribe(System.out::println, Throwable::printStackTrace);
~~~

有时候我们也会对一些可能产生错误的 Observable 使用 flatMap。RxJava 在发生 onError 之后的默认行为是立即终止整个数据流。这种处理方式的问题在于，如果我们不想使用 `onErrorXXX` 操作符（因为这样我们就无法知道是哪个数据源发生了错误），那产生错误的数据源会导致其他数据源的努力都白费了。

解决方案就是把错误事件延迟到所有的数据源均结束之后，再发出这个错误事件。这让我们可以对 flatMap 的输出应用一个“全局”的错误处理器，但又同时可以处理其他的成功事件。

因此 flatMap 有一个重载版本，在 mapper 函数后有一个 `delayErrors` 参数。merge 则没有这样的重载，而是有一个单独的函数名：`mergeDelayError`。

**Backpressure** 是防止响应式数据流的缓冲区爆炸的方式，也是 RxJava 的基石。绝大多数内部不考虑时间因素的操作符，都遵守并应用了 backpressure。但不幸的是，flatMap 只能说它遵守了 backpressure，但并没有对它的输入数据流应用 backpressure。

这就意味着如果我们使用 flatMap 和 merge 最常用的重载版本，它们都会向上游请求 `Long.MAX_VALUE` 个数据，并一次性处理所有到来的数据。无限的缓冲行为就导致了我们会保持无限个对内部产生的 Observable 的活跃订阅。

如果上游数据流长度很短，或者很低频，这个特性倒也不会产生什么问题，但如果 flatMap 的下游有一个有界的异步操作符，例如 `observeOn`，那数据就很容易在 flatMap 操作符里堆积，并造成相当大的性能下降。

从技术上来讲，我们将会看到，并没有任何理由阻止 flatMap 向上游应用同样的 backpressure 策略。但由于历史原因，有些 Rx.NET 的场景依赖于这种不考虑 backpressure 的策略，他们很乐意一次性合并 1000 数量级的 Observable，而同时又有一些场景依赖于这些数据源在合并时都保持活跃（被订阅）。因此，这一无尽缓冲的 backpressure 策略就被 RxJava 沿用了下来。

然而，很多人意识到合并 1000 个 hot Observable 和合并 1000 个 cold Observable 是两码事，因此我们需要能限制同时订阅的 Observable 的数量。因此 flatMap 和 merge 也就有了接收 `maxConcurrency` 参数的重载版本，用来实现这一限制。而一个有趣的事实是，在一个考虑了 backpressure 的环境中实现这一限制，比在没有 backpressure 的 Rx.NET 中，要简单得多。

## 实现 flatMap

考虑到上面 flatMap 和 merge 的转换，我们可能会考虑应该实现哪个操作符。显然 merge 不需要处理 mapper 函数，那何不像 RxJava 那样实现 merge？

答案是：内存分配。如果我们用 merge 实现 flatMap，我们就需要使用 merge 和 map 这两个操作符。当数据流被合并的时候，每使用一个操作符都会带来内存分配的开销。这一点无法避免，因为操作符都需要保存一些状态：参数，回调函数等等。使用更多的操作符意味着会有更多分配开销，更多的内存垃圾，以及更多的 GC 抖动，尤其是这些数据流的生命周期都很短的时候。

（_由于 RxJava 早些时候出于便利性而做出的一个决定，情况变得更糟了：引入了 `lift()` 操作符，而且它被普遍使用于标准操作符的实现中。所以应用一个操作符可能会分配 6~10 个对象，而理论上我们只需要 1~2 个即可。_）

由于 flatMap 需要把多个数据源的事件串行化，所以我们可能首先会想到实现一个 `SerializedSubscriber`。把所有的数据源都订阅到这个对象上，并把事件串行化，这应该很简单。

不幸的是，这做不到。

**首先**，把多个数据源订阅到同一个有状态的 SerializedSubscriber 对象上，是个坏主意。我们无法处理请求，而且不同的 Observable 通常都会为它们的 Subscriber 设置自己的 Producer，所以如果使用同一个 SerializedSubscriber 对象，Observable 们会覆盖各自的 Producer。

其次，我们需要移除掉终止的数据源，防止永久地持有它们的引用。由于我们没有和 `Subscriber.add()` 对应的 `Subscriber.remove()` 函数，所以即便我们用不同的 Subscriber 实例，并把事件发往同一个内部的 SerializedSubscriber，我们也需要实现类似于 CompositeSubscription 的逻辑，用于处理下游的取消订阅和清理工作。

最后，SerializedSubscriber 可能会由于 synchronized 代码块造成阻塞。阻塞天生就能带来 backpressure 支持，但同时也会方案进度。如果我们运行在异步的环境中，那我们会希望尽可能避免阻塞。

因此，实现串行化并且避免阻塞的办法，就是使用我们熟悉的队列漏了。所以让我们从 flatMap 操作符的结构开始：

~~~ java
public final class OpFlatMap<T, R> implements Operator<R, T> {
 
    final Func1<? super T, ? extends Observable<? extends R>> mapper;
     
    final int prefetch;
 
    public OpFlatMap(Func1<? super T, ? extends Observable<? extends R>> mapper,
            int prefetch) {
        this.mapper = mapper;
        this.prefetch = prefetch;
    }
     
    @Override
    public Subscriber call(Subscriber<? super R> t) {
        FlatMapSubscriber<T, R> parent = new FlatMapSubscriber<>(t, mapper, prefetch);
        parent.init();
        return parent;
    }
}
~~~

操作符接收一个 mapper 函数，用来生成内部的 Observable，以及一个 prefetch 参数，表示订阅到内部的 Observable 时初始请求的数量。我们会在 parent Subscriber 中处理这些数据以及下游 Subscriber。为了方便，设置取消订阅链以及 backpressure 封装在了 `init()` 函数中。

接下来就是用来进行协调和数据收集的 FlatMapSubscriber 了：

~~~ java
static final class FlatMapSubscriber<T, R> extends Subscriber<T> {
    final Subscriber<? super R> actual;
     
    final Func1<? super T, ? extends Observable<? extends R>> mapper;
     
    final int prefetch;                                             // (1)
 
    final CompositeSubscription csub;                               // (2)
     
    final AtomicInteger wip;                                        // (3)
     
    final Queue<Object> queue;                                      // (4)
     
    final AtomicLong requested;                                     // (5)
 
    final AtomicInteger active;                                     // (6)
     
    final AtomicReference<Throwable> error;                         // (7)
     
    public FlatMapSubscriber(Subscriber<? super R> actual,
            Func1<? super T, ? extends Observable<? extends R>> mapper,
            int prefetch) {
        this.actual = actual;
        this.mapper = mapper;
        this.prefetch = prefetch;
        this.csub = new CompositeSubscription();
        this.wip = new AtomicInteger();
        this.requested = new AtomicLong();
        this.queue = new ConcurrentLinkedQueue<>();
        this.active = new AtomicInteger(1);
        this.error = new AtomicReference<>();
    }
     
    public void init() {
        // TODO implement
    }
     
    @Override
    public void onNext(T t) {
        // TODO implement
    }
     
    @Override
    public void onError(Throwable e) {
        // TODO implement
    }
     
    @Override
    public void onCompleted() {
        // TODO implement
    }
     
    void childRequested(long n) {
        // TODO implement
    }
 
    void innerNext(Subscriber<R> inner, R value) {
        // TODO implement
    }
     
    void innerError(Throwable ex) {
        // TODO implement
    }
     
    void innerComplete(Subscriber<?> inner) {
        // TODO implement
    }
     
    void drain() {
        // TODO implement
    }
}
~~~

到目前为止都还没有特殊的地方，就是些寻常的成员和参数：

1. 我们需要保存 child Subscriber，mapper 函数，以及 prefetch 参数；
2. 我们用 CompositeSubscription 来保存内部的 Subscriber，这样当下游取消订阅时，我们可以一次性取消所有的 Subscriber，而当某个 Observable 终止时，我们也可以移除它的 Subscriber；
3. 然后就是标记是否有线程在漏循环中的 wip 标记了，利用它我们将实现非阻塞的队列漏模型；
4. 我们用一个共享的队列，来保存所有 Observable 发出的数据，以供漏循环中发往下游；队列容纳的是 Object 而非 R 类型，因为我们还需要把产生数据的 Observable 对应的 Subscriber 也存到队列中，这样我们就能向这个数据源请求更多的了；
5. 我们还需要保存下游发出的请求量，因为如果下游请求了 1，我们无法确认该向哪个上游发出这个请求，所以我们只能向所有的上游都发出请求；然而这样做可能会导致上游产生任意多的数据，我们也不能一次性把这些数据都发往下游（有可能会在下游触发 `MissingBackpressureException`）；
6. 我们也需要记录当前有多少个活跃的上游，包括主要的 T 类型上游；当这个计数到 0 时，我们就可以终止掉下游了；
7. 任何一个上游，包括最初的上游，都可能发出错误；为了方便，我们只会保存第一个发出的错误，其他的都会交给 RxJavaPlugins 的错误处理器；

大家可能会想，如果存在（5）中所说的**请求放大**，为什么不逐个请求呢？原因有二：a) 逐个请求对大多数数据源来说都相当低效；b) 即便逐个请求，每次我们还是会收到 N 个数据（因为会每个数据源都请求 1 个数据），我们还是得做些计数工作，以确定何时再向内部的数据源发出请求。

在我们实现上面留空的函数之前，我们还需要一个类：`FlatMapInnerSubscriber`，它用来订阅 mapper 函数产生的每一个 `Observable<R>`。由于我们不能继承两个类，或者实现两个类型参数不一样的同一接口，所以我们需要这个单独的类：

~~~ java
static final class FlatMapInnerSubscriber<T, R> extends Subscriber<R> {
    final FlatMapSubscriber<T, R> parent;
 
    public FlatMapInnerSubscriber(FlatMapSubscriber<T, R> parent, int prefetch) {
        this.parent = parent;
        request(prefetch);                                         // (1)
    }
     
    @Override
    public void onNext(R t) {
        parent.innerNext(this, t);                                 // (2)
    }
     
    @Override
    public void onError(Throwable e) {
        parent.innerError(e);
    }
     
    @Override
    public void onCompleted() {
        parent.innerComplete(this);
    }
 
    void requestMore(long n) {
        request(n);                                                // (3)
    }
}
~~~

这里我们起始请求 `prefetch` 个数据（1），并把所有的 onXXX 事件转发给外部 FlatMapSubscriber。在外部的 FlatMapSubscriber 中，我已经提到，我们会把数据连同数据发出者一起加入到共享的 `Queue<Object>` 中，这一点可能并不直观，而且可能有人会在（2）的前后调用 `request(1)`。这样做的问题是这个上游就会持续发出数据，淹没队列，进而完全失去 backpressure 的效果。解决办法就是只有当数据源产生的数据被实际消费（发往下游）时，才请求下一个数据。（在第二部分中，我们会用另一种方式来解决这个问题）。此外，我们还需要把 protected 的 request 方法暴露出去，以允许在漏循环中请求更多数据。

好了，现在回到 FlatMapSubscriber：

~~~ java
public void init() {
    add(csub);
    actual.add(this);
    actual.setProducer(new Producer() {
        @Override
        public void request(long n) {
            childRequested(n);
        }
    });
}
~~~

在 init() 函数中，我们设置好取消订阅链条，并把下游的 request() 函数转发到我们的 childRequested() 函数中。大家可能会问，为什么不在构造函数中做这件事？把它们拆为两个函数，我们就不会在构造函数中发生“this 指针逃逸”的问题。

~~~ java
@Override
public void onNext(T t) {
    Observable<? extends R> o;
     
    try {
        o = mapper.call(t);
    } catch (Throwable ex) {
        Exceptions.throwOrReport(ex, this, t);
        return;
    }
     
    active.getAndIncrement();
    FlatMapInnerSubscriber<T, R> inner = 
            new FlatMapInnerSubscriber<>(this, prefetch);
    csub.add(inner);
     
    o.subscribe(inner);
}
~~~

在 onNext() 函数中，我们调用 mapper 函数来生成一个 Observable，递增 active 计数器，创建 FlatMapInnerSubscriber，把它加入到容器中，**最后**才订阅到生成的 Observable 上。这样，当 Observable 终止时，我们不会把 active 计数器递减到 0，而且也可以把它自己从容器中移除。由于 mapper 函数可能会抛出异常，所以我们用 try-catch 包裹起来，并把利用辅助函数把异常抛出（例如 OutOfMemoryError，StackOverflowError），或者通过 RxJava 的机制进行通知（例如 onError）。

~~~ java
@Override
public void onError(Throwable e) {
    if (error.compareAndSet(null, e)) {
        unsubscribe();
        drain();
    } else {
        RxJavaPlugins.getInstance()
        .getErrorHandler().handleError(e);
    }
}
~~~

我们尝试为唯一的 error 成员设置赋值，如果成功我们就取消订阅自己（进而会取消订阅所有的 FlatMapInnerSubscriber），并调用 drain() 函数，其中我们会在保证串行访问的前提下把错误发往下游。如果此前已经发生了错误，那我们就把错误交给 RxJavaPlugins 的错误处理器。这里我们无需考虑 active 计数器，因为 onError 对我们来说是一个立即的终止信号，这和 onCompleted 不同。

~~~ java
@Override
public void onCompleted() {
    if (active.decrementAndGet() == 0) {
        drain();
    }
}
~~~

这里我们会递减 active 计数器，如果递减到 0，我们就调用 drain() 函数。drain() 函数会保证队列里所有的数据发完之后再发出 onCompleted 事件。由于我们把初始的上游也当做输入，所以 active 计数器从 1 开始，并随着内部 Observable 的订阅和终止而增减。这里我们依赖于内部生成的 Observable 遵守游戏规则，至多产生一次终止事件。那么如果初始上游终止时，active 计数器为 1，我们就可以确认没有内部的 Observable 还处于活跃状态了（后面也不会有 0-1-0 的变化了）。如果我们无法确信上游都遵守契约，那我们可以用 CAS 操作来保护计数器，确保每个上游只会递减一次：

~~~ java
AtomicBoolean once = new AtomicBoolean();
// ...
@Override
public void onCompleted() {
    if (once.compareAndSet(false, true)) {
        if (active.decrementAndGet() == 0) {
            drain();
        }
    }
}
~~~

由于 RxJava 的设计决策，FlatMapSubscriber 不能实现 Producer 接口，因此我们需要另外一个单独的 Producer 对象，就像在 init() 函数里一样。childRequested 的代码如下：

~~~ java
void childRequested(long n) {
    if (n > 0) {
        BackpressureUtils.getAndAddRequest(requested, n);
        drain();
    }
}
~~~

辅助函数确保了请求计数被限定在 `Long.MAX_VALUE` 以内，然后我们会调用 drain() 函数。

FlatMapInnerSubscriber 调用的委托方法比较简单，这里我就一起展示了：

~~~ java
void innerNext(Subscriber<r> inner, R value) {
    queue.offer(inner);
    queue.offer(NotificationLite.instance().next(value));
    drain();
}
    
void innerError(Throwable ex) {
    onError(ex);
}
    
void innerComplete(Subscriber<r> inner) {
    csub.remove(inner);
    onCompleted();
}
~~~

innerNext() 中我们把 subscriber 以及数据都放进队列中（通过一层包装确保 null 可以放进队列），然后调用 drain 函数；innerError 则是把异常转发给 onError，最后，innerComplete 把 subscriber 从容器中移除，并调用 onCompleted。注意，要是你实现了前面我提到的单次终止保障，这里我们就不能这样直接转发，你还得在 FlatMapInnerSubscriber 里也实现一样的单次保障，确保 innerCompleted 只被调用一次，并在 innerCompleted 里检查 decrementAndGet() == 0。

最后，让我们逐行解析 drain 函数：

~~~ java
if (wip.getAndIncrement() != 0) {
    return;
}
 
int missed = 1;
 
for (;;) {
     
    long r = requested.get();
    long e = 0L;
     
    while (e != r) {
~~~

第一部分很典型，我们递增 wip 计数器，如果递增前是 0，我们就进入漏循环。我们用 missed 计数器来检测是否有其他线程也调用了 drain 函数，因此我们可以继续多做些事情。我们获取当前的请求计数，准备好发射计数。循环会一直持续到发射计数达到请求计数。

~~~ java
        if (actual.isUnsubscribed()) {
            return;
        }
        
        boolean done = active.get() == 0;              // (1)
        Throwable ex = error.get();                    // (2)
        if (ex != null) {
            actual.onError(ex);
            return;
        }
        
        Object o = queue.poll();
        
        if (done && o == null) {                       // (3)
            actual.onCompleted();
            return;
        }
        
        if (o == null) {
            break;
        }
    
        Object v;
        
        for (;;) {                                     // (4)
            if (actual.isUnsubscribed()) {
                return;
            }
            v = queue.poll();
            if (v != null) {
                break;
            }
        }
        
        actual.onNext(NotificationLite
                .<R>instance().getValue(v));           // (5)
        
        ((FlatMapInnerSubscriber<?, ?>)o)
            .requestMore(1);
        
        e++;
    }
~~~

1. 这个漏循环里除了内层循环看起来都应该很熟悉了；在 while 循环里，我们通过检查 active 计数器来确定是否结束；
2. 如果 error 成员不为 null，我们也认为已经结束；
3. 由于我们一次放了两个对象到队列中，我们也得一次取出两个对象；如果第一个取出来为 null，就说明队列已经空了，而如果此时 done 了，那我们也就确认上游已经终止了，所以可以给下游发送 onCompleted 了；
4. 但我们不能直接 pull 第二次，因为执行 innerNext() 的线程可能在两次 offer() 之间被中断，因此第二个要放进队列的对象（实际发出的数据）还没有放进来；所以我们需要一个内层循环不停 pull，知道取出数据（或者下游取消订阅）；（注意内层循环可以通过一些特殊的队列或者元组类型避免）
5. 有了实际数据之后，我们就可以拆包了（利用 NotificationLite），然后把它发往下游；与之同时，我们把 subscriber 强转为 FlatMapInnerSubscriber，并请求下一个数据；通过递增发射计数器，我们可以退出主循环体，表明我们已经满足了所有的请求量；

~~~ java
    if (e == r) {
        if (actual.isUnsubscribed()) {
            return;
        }
        boolean done = active.get() == 0;
        Throwable ex = error.get();
        if (ex != null) {
            actual.onError(ex);
            return;
        }
         
        if (done && queue.isEmpty()) {
            actual.onCompleted();
            return;
        }
    }
     
    if (e != 0L) {
        BackpressureUtils.produced(requested, e);
    }
     
    missed = wip.addAndGet(-missed);
    if (missed == 0) {
        break;
    }
}
~~~

最后一个部分处理了所有的请求都已被发射，只需要终止事件流的情况。此外，如果所有的内部 Observable 都是空的，而下游又没有请求任何数据，也可能 `e == r`；这里的逻辑确保了我们会激进地向下游发出终止事件。如果发出了事件，那我们就利用另一个辅助函数把请求计数器减去发射数量。然后我们递减 wip 计数器，如果递减为 0，则说明我们可以退出漏循环了。否则我们将继续循环（带着新的 missed 计数器）。

## 总结

如果你看看 RxJava 1.x 中 merge 的实现，以及 RxJava 2.x 或者 Reactor 2.5 中 flatMap 的实现，你会发现它们的实现和我上面展示的有很大的差别。差别的原因在下篇中我们将要讲解的性能和功能考虑。

你可能会想，为什么不直接讲解 RxJava 1.x 的实现呢？原因有二：复杂度，基础代码结构。我认为，在本篇中的实现对那些一直阅读本博客的朋友更容易理解，因为都基于以前讲过的一些概念，而基于此，也更容易理解下篇中我们将要讲解的内容。
