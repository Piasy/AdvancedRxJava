---
layout: post
title: 操作符熔合（二）
tags:
    - Operator
---

原文 [Operator-fusion (part 2 - final)](http://akarnokd.blogspot.com/2016/04/operator-fusion-part-2-final.html){:target="_blank"}。

## 介绍

在上篇中，我介绍了操作符熔合的一些相关概念。在本文中，我将详细讲解实现操作符熔合的 API 和协议。

当前的操作符熔合都只适用于相邻的两个操作符，且基于双方都能互相知晓对方类型的前提，而在微熔合中，如果双方都同意新的协议，那将忽略 Reactive-Streams（RS） 规范而是用新的协议。

## 宏熔合的结构

宏熔合的主要目标是那些单一元素的操作符：`just()`，`empty()`，`fromCallable()`。为了单一的元素而创建整个 RS 的基础设施，开销着实太大，而且 RxJava 和 Reactor 项目中操作符的使用情况中，超过半数都是这些操作符。因此 RxJava 引入了 `Single` 类，Reactor 引入了 `Mono` 类，来尽可能降低这些需求下的开销，而且还为这两个类提供了更加优化的操作符版本。

但是如果我们能在装配期知道一个数据源最多发出一个数据，那对普通 `Observable`/`Flux` 的使用也能带来很大优化。此外，知晓数据源的情况肯定也有助于我们通过自定义操作符来内联多个操作符。

### 至多产生一个数据的同步数据源

为了标识数据源至多只发出一个数据，[Reactive-Streams-Commons](https://github.com/reactor/reactive-streams-commons)（Rsc）项目中（以及 Reactor）设计了一个协议：

***如果 `Publisher` 实现了 `java.util.concurrent.Callable` 接口，那就表明它至多发出一个数据。*

所以我们可以实现 `Callable` 接口，并在 `call()` 函数中返回一个可以同步计算的非 null 值，或者返回 null 表示没有数据。注意，RS 并不允许在 onNext 中传递 null。`call()` 函数将在订阅期被调用。

~~~ java
public class MySingleSource implements Publisher<Object>, Callable<Object> {
    @Override
    public void subscribe(Subscriber<? super Object> s) {
        s.onSubscribe(new ScalarSubscription<>(s, System.currentTimeMillis()));
    }
 
    @Override
    public Object call() throws Exception {
        return System.currentTimeMillis();
    }
}
~~~

如果我们知道这个要被发出的数据是固定的，那这个数据源就可以应用装配期优化了。例如，如果 `call()` 返回 null，表明这个数据源没有数据（例如 `empty()`），那就只有少量的操作符可以应用（那些不作用与数据上的操作符），那装配的过程就可以一直返回 `empty()`。

**我们可以扩展 `Callable` 为一个新的 `ScalarCallable` 接口，来表示至多发出一个固定不变的数据。**

~~~ java
public interface ScalarCallable<T> extends Callable<T> {
    @Override
    T call();
}
~~~

通过继承自 `Callable`，任何期待动态单一数据的操作符，都可以接受固定单一数据的数据源。当然反过来并不成立，期待固定单一数据的操作符，不会在装配期执行其他类型的 `Callable`（因为可能阻塞，或者触发其他副作用）：

~~~ java
public class MyScalarSource implements Publisher<Object>, ScalarCallable<Object> {
    @Override
    public void subscribe(Subscriber<? super Object> s) {
        s.onSubscribe(new ScalarSubscription<>(s, 1));
    }
 
    @Override
    public Object call() {
        return 1;
    }
}
~~~

注意 `ScalarCallable` 重写了 `call()` 函数，并去掉了异常列表：标量常量是不应该抛出异常的，因此消费者也就不需要把 `call()` 用 try-catch 包起来。

### 至多消费一个数据的同步数据源

消费 `Callable` 和 `ScalarCallable` 就是在订阅期或者装配期使用 `instanceof` 检查类型，然后通过 `call()` 取出这个唯一的数据。

例如，对 `count()` 操作符的宏熔合就可以检查单一数据的数据源，如果数据源是空的，那就可以直接返回 0，如果有一个数据，那就可以直接返回 1：

~~~ java
public final Flux<Long> count() {
    if (this instanceof ScalarCallable) {
 
       T value = ((ScalarCallable<T>)this).call();
 
       return just(value == null ? 0 : 1);
    }
    return new FluxCount<>(this);
}
~~~

另一个例子是在 `flatMap()`，`concatMap()` 或者 `switchMap()` 中为单一数据的数据源创建一个短路逻辑。在这种情况下也就没必要创建完整的基础设施，直接订阅 mapper 函数返回的 `Publisher` 即可。

注意，由于 mapper 函数可能自己会有副作用，我们不能使用装配期的优化，因此我们可以引入一个新的类型：

~~~ java
public final <R> Px<R> flatMap(
        Function<? super T, ? extends Publisher<? extends R>> mapper) {
 
    if (this instanceof Callable) {
 
        return new PublisherCallableMap<((Callable<T>)this, mapper);
    }
 
    return new PublisherFlatMap<>(this, mapper, ...);
}
~~~

（注：`Px` 在 Rsc 中表示 **Publisher Extensions**，是 Rsc 链式 API 的基础类，起这个名字更大程度上是为了方便单元测试和性能测试时缩短拼写）

~~~ java
public final class PublisherCallableMap<T, R> implements Publisher<R> {
    final Callable<? extends T> source;
    final Function<? super T, ? extends Publisher<? extends T>> mapper;
 
    public PublisherCallableMap(
            Callable<? extends T> source,
            Function<? super T, ? extends Publisher<? extends T>> mapper) {
        this.source = source;
        this.mapper = mapper;
    }
 
    @Override
    public void subscribe(Subscriber<? super R> s) {
        T value;
 
        try {
            value = source.call();                                    // (1)
        } catch (Throwable ex) {
            ExceptionHelper.throwIfFatal(ex);
            EmptySubscription.error(s, ex);
            return;
        }
 
        if (value == null) {
            EmptySubscription.complete(s);
            return;
        }
 
        Publisher<? extends R> p;
 
        try {
            p = mapper.apply(value);                                  // (2)
        } catch (Throwable ex) {
            ExceptionHelper.throwIfFatal(ex);
            EmptySubscription.error(s, ex);
            return;
        }
 
        if (p == null) {
            EmptySubscription.error(s, 
                new NullPointerException("The mapper returned null");
            return;
        }
 
        if (p instanceof Callable) {                                  // (3)
            R result;
 
            try {
                result = ((Callable<R>)p).call();
            } catch (Throwable ex) {
                ExceptionHelper.throwIfFatal(ex);
                EmptySubscription.error(s, ex);
                return;
            }
 
            if (result == null) {
                EmptySubscription.complete(s);
                return;
            }
 
            s.onSubscribe(new ScalarSubscription<>(s, result));
 
            return;
        }
 
        p.subscribe(s);
    }
}
~~~

首先我们从 `Callable` 中取出唯一的数据（1）。如果是 null，那就可以直接结束 `Subscriber` 了。否则我们调用 mapper 函数得到 `Publisher`（2）。由于这个 `Publisher` 也可能是一个 `Callable`，所以我们也对其进行检查（3），然后如果这个 `Callable` 返回 null 那就直接终止，否则就把一个支持 backpressure 的 `ScalarSubscription` 设置给 `Subscriber`。由于 `call()` 可能抛出异常，所以我们捕获所有的异常，并把致命的错误按照库特定的方式抛出，不致命的异常则通过 onError 抛出（同时在正确的时机设置 `Subscription`）。

### 使用 Callable 时要小心

由于 Callable 并不是一个标记接口，我们必须小心 `Publisher` 和 `Callable` 的实现者，因为 `Callable` 功能上并不是用来表示至多发出一个数据的。

对此我的希望是，由于 RS 还是个新事物，很少有人用它实现了自己的操作符，所以我们可以避免这种接口组合方式导致的坑。

## 微熔合的结构

和宏熔合不一样，微熔合相邻的两个操作符做一个协议的切换，RS 规范定义的接口被部分或者全部替换为协议定义的接口。这样就允许我们在操作符之间共享数据结构或者状态了。

理论上来说，一对操作符中，上游操作符可以负责初始化工作，并利用下游操作符的内部资源。但在实际情况中，到目前为止我们都是反其道而行之：下游操作符使用上游操作符的内部资源。

然而我们也不建议实现完整的自定义交互，因为这会导致彻底的自定义实现，以及大量重复的代码。（但不得不承认，`ConditionalSubscriber` 需要重复代码以避免强转。）

目前 Rsc 和 Reactor 实现了两种微熔合：条件式和基于队列的熔合。从第二个维度来看，我们可以考虑三种操作符：

+ 支持熔合的数据源（`range()`，`UnicastProcessor`）
+ 可能支持熔合的中间操作符（`concatMap`，`observeOn`，`groupBy`，`window`）
  - 头部熔合（`concatMap`）
  - 尾部熔合（`groupBy`）
  - 过渡熔合（`map`，`filter`）
+ 消费者（`flatMap` 的内部订阅者，`zip`）

基于队列的熔合还有第三个维度：数据源是同步的（`fromArray`）还是异步的（`UnicastProcessor`）。

### 条件式熔合

条件式微熔合的标志是 `ConditionalSubscriber` 接口，它扩展自 `Subscriber` 接口：

~~~ java
public interface ConditionalSubscriber<T> extends Subscriber<T> {
    boolean onNextIf(T t);
}
~~~

如果数据源或者中间操作符发现它的下游是 `ConditionalSubscriber`，那它就可能调用 `onNextIf` 方法。（由于函数调用天生就是同步的，所以条件式熔合只适用于同步场景）

如果 `onNextIf` 返回 true，说明这个数据被整除消费。如果返回 false，则说明这个数据被丢弃，可以立即新发一个数据。这一机制避免了 `filter` 等操作符补充数据时大量的 `request(1)` 调用。

_注：大家可能会问，为什么这个优化很重要？因为 `request()` 调用通常都需要一次 CAS 操作，相当于每个被丢弃的数据会导致 21~45 个 CPU 时钟周期的开销。_

为了在数据源操作符中使用 `ConditionalSubscriber`，我们需要在订阅期检查 `Subscriber` 的类型，来使用不同的实现，以避免对下游 Subscriber 一直做强转。

~~~ java
@Override
public void subscribe(Subscriber<? super Integer> s) {
    if (s instanceof ConditionalSubscriber) {
 
        s.onSubscribe(new RangeConditionalSubscription<>(
            (ConditionalSubscriber<T>)s, start, count));
 
    } else {
        s.onSubscribe(new RangeSubscription<>(s, start, count);
    }
}
~~~

在单独的实现中我们就可以在发射数据时使用 `onNextIf` 了。例如快路径可以改写为如下形式：

~~~ java
for (long i = start; i < (long)start + count; i++) {
    if (cancelled) {
        return;
    }
    s.onNextIf((int)i);
}
if (!cancelled) {
    s.onComplete();
}
~~~

大家可能会想，既然不关心 `onNextIf` 的返回值，那为什么还要调用它呢？这是出于组合的考虑。即便在快路径上我们不需要返回值，但如果下游也调用 `onNextIf`，这就能避免整个不必要的 `request(1)` 调用链了。

相较而言慢路径更有趣一些：

~~~ java
long i = index;
long end = (long)start + count;
long r = requested;
long e = 0L;
 
while (i != end && e != r) {
    if (cancelled) {
       return;
    }
     
    if (s.onNextIf((int)i)) {
        e++;
    }
    i++;
}
 
if (i == end) {
    if (!cancelled) {
        s.onComplete();
    }
    return;
}
 
if (e != 0L) {
    index = i;
    REQUESTED.addAndGet(this, REQUESTED, -e);
}
~~~

在循环中，如果 `onNextIf` 返回 false，我们就不递增发射计数器，这意味着我们可以立即发出下一个数据。如果下游消费者请求了 1 个数据，但把所有的数据都丢弃掉，那上面的循环将会耗尽所有可用的数据，一次原子 `addAndGet` 都不会调用。

由于 filter 是非常常用的操作符，所以每个操作符即便不影响流过的数据量，也应该保持对 `ConditionalSubscriber` 的支持。例如 `map()` 和 filter 一起出现，因此我们也需要 `map()` 支持条件式熔合，检查 `Subscriber` 的类型，应用不同的实现：

~~~ java
static final class MapConditionalSubscriber<T, R> implements ConditionalSubscriber<T> {
    final ConditionalSubscriber<? super R> actual;
     
    final Function<? super T, ? extends R> mapper;
 
    boolean done;
 
    Subscription s;
 
    // ...
 
    @Override
    public boolean onNextIf(T t) {
        if (done) {
            return;
        }
 
        R v;
         
        try {
            v = mapper.apply(t);
        } catch (Throwable ex) {
            ExceptionHelper.throwIfFatal(ex);
            s.cancel();
            onError(ex);
            return;
        }
 
        if (v == null) {
            s.cancel();
            onError(new NullPointerException("..."));
            return;
        }
 
        return actual.onNextIf(v);
    }
 
    // ...
}
~~~

条件式微熔合最后的场景就是“终点”操作符，或者说消费者了。幸运的是，我们通常不需要两种单独的实现，`ConditionalSubscriber` 和 `Subscriber`，而是可以同时实现它们。那些支持 `ConditionalSubscriber` 的消费者可以支持它，其他的使用普通的 `Subscriber` 即可：

~~~ java
static final FilterSubscriber<T> implements ConditionalSubscriber<T> {
    final Subscriber<? super T> actual;
 
    final Predicate<? super T> predicate;
 
    boolean done;
 
    Subscription s;
 
    // ...
 
    @Override
    public void onNext(T t) {
 
        if (!onNextIf(t)) {
           s.request(1);
        }
    }
 
    @Override
    public boolean onNextIf(T t) {
        if (done) {
            return;
        }
         
        boolean pass;
 
        try {
            pass = predicate.test(t);
        } catch (Throwable ex) {
            ExceptionHelper.throwIfFatal(ex);
            s.cancel();
            onError(ex);
        }
 
        if (pass) {
            actual.onNext(t);
            return true;
        }
        return false;
    }
 
    // ...
}
~~~

总结一下，条件式微熔合相对而言比较简单，但也比较繁琐，它能省去 `request(1)` 的调用，进而降低每个元素发射时的开销。

### 基于队列的熔合

这是目前为止在响应式编程领域内最复杂的话题，不是因为它需要多么复杂的数据结构和算法，而是因为操作符组合的组合数爆炸，对每个 op1 后面的 op2，我们都需要判断能否支持熔合。

基于队列的熔合工作的基础是很多操作符都会有一个队列，用来实现 backpressure 或者异步的支持，而这些操作符之间的队列会进行数据转移。

例如，`UnicastProcessor` 有个尾部队列，会把数据保存起来等待下游请求，而 `concatMap` 有个头部队列，用来保存将要转换为 `Publisher` 的数据。订阅时数据从一个队列转移到另一个队列，形成一组 `dequeue`-`enqueue` 调用，除了带来原子操作、请求管理和计数的开销，没有任何功能性。

显然，如果我们能通过某种方式在两个操作符之间共用一个队列，以及降低队列原子操作的开销，这样我们就能在计算量和内存分配上降低不少开销。

但是，如果中间的操作符对数据做了操作，怎么办？如果这种情况下不应该熔合，怎么办？

为了解决这个协调的问题，我们可以复用 RS 中已有的 `onSubscribe(Subscription)`，并且扩展这一协议，这便是 `QueueSubscription`。

~~~ java
public interface QueueSubscription<T> extends Queue<T>, Subscription {
 
    int NONE = 0;
    int SYNC = 1;
    int ASYNC = 2;
    int ANY = SYNC | ASYNC;
    int THREAD_BOUNDARY = 4;
 
    int requestFusion(int mode);
 
    @Override
    default boolean offer(T t) {
        throw new UnsupportedOperationException();
    }
 
    // ...
}
~~~

`QueueSubscription` 接口是对 `Queue` 和 `Subscription` 的组合，增加了一个 `requestFusion()` 方法，而从父接口继承的方法都改成了默认抛出 `UnsupportedOperationException`。（Java 7 注意：如果你的类无法继承自一个做了这一操作的基类，那就只能手动做这一操作）

+ `void request(long n)`
+ `void cancel()`
+ `T poll()`
+ `boolean isEmpty()`
+ `void clear()`

有的库可能也会实现 `size()` 接口，用于排查错误。

如果数据源支持队列熔合，它就可以在 `onSubscribe` 中传出一个 `QueueSubscription`。那些接受这一协议的操作符就可以对它进行操作，其他的则只把它当做普通的 `Subscription`。

那些接受这一协议的操作符就可以把它当做 Queue 来使用，从而不用创建自己的队列，以节省内存分配和开销。此外，诸如 `range()` 的数据源可以把自己伪装成一个队列，通过 `poll()` 接口返回下一个数据，没有数据时返回 null。

由于有些情况下我们无法应用也不应该应用熔合，我们需要在订阅期做一次协议切换。这一切换可以通过 `requestFusion()` 完成。

（_注：我知道 enum 可读性更好，但 `EnumSet` 也存在不小的开销。_）

输入参数可选：

+ `SYNC`：表明消费者希望和同步的上游一起工作，通常都是知道数据量的；
+ `ASYNC`：表明消费者希望和异步的上游一起工作，而且通常都是不知道数据量和数据发射时机的；
+ `ANY`：表明消费者可以支持 `SYNC` 和 `ASYNC`；
+ `(SYNC, ASYNC) | THREAD_BOUNDARY`：表明消费者跨越了线程边界，`poll()` 可能从其他的线程调用；

返回值可选：

+ `NONE`：不支持熔合；
+ `SYNC`：已启用同步熔合模式；
+ `ASYNC`：已启用异步熔合模式；

如果上游不支持请求的熔合模式，或者对线程边界很敏感，它就可以返回 `NONE`。这时数据流将按照标准的 RS 规范运行。（注意，条件式熔合仍有可能发生）

由于熔合是可选的，协商成功的模式需要一端或者两端都支持不同的运行模式。此外，模式切换必须在任何事件之前完成，因此 `onSubscribe` 是一个最佳的时机，因为 RS 规范规定了此前不允许任何事件发生。

`SYNC` 和 `ASYNC` 都有需要实现者额外遵守的要求。

在 `SYNC` 模式下，消费者不允许调用 `request()`，生产者在 `poll()` 中不允许返回 null，除非是为了表明终止事件。由于所有的交互只有 `poll()` 和 `isEmpty()`，因此数据源没有机会调用 `onError`，只能在上述方法中抛出异常。另一方面，消费者也就必须用 try-catch 包住上述方法调用，并处理异常。

在 `ASYNC` 模式下，生产者把事件放进自己的队列中，然后通知消费者来取数据。最合适的时机是 `onNext`。我们可以发出数据本身或者 null（只有这里允许使用 null）。在消费者这边，`ASYNC` 模式下的 `onNext` 传入的数据是无意义的，应该被忽略。其他的方法，`onError`，`onComplete`，`request` 和 `cancel` 应该按照 RS 规范使用。在异步模式下，`poll()` 可以返回 null，表示暂时没有数据，终止事件将通过 `onError` 和 `onComplete` 发出。

#### 实现支持熔合的数据源

现在让我们看看实际的 API，首先是让 `range()` 支持熔合：

~~~ java
static final class RangeSubscription extends QueueSubscription<Integer> {
     
    // ... the Subscription part is the same
 
    @Override
    public Integer poll() {
        long i = index;
        if (i == (long)start + count) {
            return null;
        }
        index = i + 1;
        return (int)i;
    }
 
    @Override
    public boolean isEmpty() {
        return index == (long)start + count;
    }
 
    @Override
    public void clear() {
        index = (long)start + count;
    }
 
    @Override
    public int requestFusion(int mode) {
        return SYNC;
    }
}
~~~

没有任何请求处理的现象，因为 `range()` 工作于**同步 pull 模式**，消费者通过在需要新数据时调用 `poll()` 函数来实现 backpressure。

相应的 `UnicastProcessor`（有点像 `onBackpressureBuffer()`）可以支持 `ASYNC` 模式的熔合：

~~~ java
public final class UnicastProcessor<T> implements Processor<T, T>, QueueSubscription<T> {
 
    volatile Subscriber<? super T> actual;
 
    final Queue<T> queue;
 
    int mode;
 
    // ...
 
    @Override
    public void onNext(T t) {
        Subscriber<? super T> a = actual;
        if (mode == ASYNC && a != null) {
            a.onNext(null);
        } else {
            queue.offer(t);
            drain();
        }
    }
 
    @Override
    public int requestFusion(int m) {
        if ((m & ASYNC) != 0) {
            mode = ASYNC;
            return ASYNC;
        }
        return NONE;
    }
 
    @Override
    public T poll() {
        return queue.poll();
    }
 
    @Override
    public boolean isEmpty() {
        return queue.isEmpty();
    }
 
    @Override
    public void clear() {
        queue.clear();
    }
 
    @Override
    public void subscribe(Subscriber<? super T> s) {
        if (ONCE.compareAndSet(this, 0, 1)) {
            s.onSubscribe(this);
            actual = s;
            if (cancelled) {
                actual = null;
            } else {
                if (mode != NONE) {
                    if (done) {
                        if (error != null) {
                            s.onError(error);
                        } else {
                            s.onComplete();
                        }
                    } else {
                        s.onNext(null);
                    }
                } else {
                    drain();
                }
            }
        } else {
            EmptySubscription.error(s, new IllegalStateException("..."));
        }
    }
}
~~~

熔合要求我们做出如下改变：

+ `onNext` 需要调用 `actual.onNext` 而不是 `drain()`；
+ `requestFusion` 中需要检查下游是否请求的是 `ASYNC`；
+ Queue 的接口需要委托到内部的 `queue` 对象；
+ `subscribe()` 也需要调用 `actual.onNext` 而不是 `drain()`；

看起来也不是很复杂，对吧？现在你可以通过实现一个支持 `SYNC` 模式的 `UnicastProcessor` 来检验一下自己对操作符熔合的理解了，什么时候可以支持，怎么支持，为什么？

#### 实现支持熔合的中间操作符

通常在实际使用中会有一些中间操作符横亘在支持熔合的数据源和消费者之间，不幸的是，这会打破我们的熔合功能（并回退到常规的 RS），甚至更糟，数据会跳过中间操作符，导致错误。

第二种问题会在操作符把自己在 `onSubscribe` 中接收到的 `Subscription` 转发出去的时候出现。想象一下如果 `map()` 这么做了，下面的序列将会输出什么？

~~~ java
range(0, 10)
    .map(v -> v + 1)
    .concatMap(v -> just(v))
    .subscribe(System.out::println);
~~~

正常情况下应该打印出 1~10，但如果 `range()` 和 `concatMap()` 都实现了熔合，且 `map()` 转发了 `Subscription`，那将打印出 0~9！这种问题可以影响到所有的操作符。

解决办法就是所有不参与熔合的操作符都不要直接转发上游的 `Subscription`，一种可能的形式就是实现自己的 `Subscription`：

~~~ java
static final class MapSubscriber<T, R> implements Subscriber<T>, Subscription {
    // ...
      
    @Override
    public void onSubscribe(Subscription s) {
        this.s = s;
 
        actual.onSubscribe(this);
    }
 
    @Override
    public void request(long n) {
        s.request(n);
    }
 
    @Override
    public void cancel() {
        s.cancel();
    }
 
    // ...
}
~~~

在实践中，很多处理了请求或者取消的操作符都会实现上面的逻辑，这里的不直观性是可以接受的，为此我们换来了更低的开销。

但这一规则影响了跨库的行为。即便不同的库使用的可能是不同的熔合协议，但它们都可能转发 `Subscription`，因此当我们跨库使用时，可能会发生前面提出的问题。通常来说，每个库都应该有一个 `hide()` 或者 `asObservable()` 函数，用来隐藏实际的数据源，也能阻止内部特性被意外传播出去。

幸运的是，`map()` 可以参与到这一熔合中来：只要它自己可熔合，调解上下游的 `requestFusion()` 函数，以及 `poll()` 函数。

~~~ java
static final class MapSubscriber<T, R> implements Subscriber<T>, QueueSubscription<R> {
    final Subscriber<? super R> actual;
     
    final Function<? super T, ? extends R> mapper;
 
    QueueSubscription<T> qs;
 
    Subscription s;
 
    int mode;
 
    // ...
 
    @Override
    public void onSubscribe(Subscription s) {
        this.s = s;
        if (s instanceof QueueSubscription) {
            qs = (QueueSubscription<T>)s;
        }
 
        actual.onSubscribe(this);
    }
 
    @Override
    public void onNext(T t) {
        if (mode == NONE) {
             
            // error handling omitted for brevity
 
            actual.onNext(mapper.apply(t));
 
        } else {
            actual.onNext(null);
        }
    }
 
    @Override
    public int requestFusion(int m) {
        if (qs == null || (m & THREAD_BOUNDARY) != 0) {
            return NONE;
        }
        int u = qs.requestFusion(m);
        mode = u;
        return u;
    }
 
    @Override
    public R poll() {
        T t = qs.poll();
        if (t == null) {
            return null;
        }
        return mapper.apply(t);
    }
 
    @Override
    public boolean isEmpty() {
        return qs.isEmpty();
    }
 
    @Override
    public void clear() {
        qs.clear();
    }
}
~~~

`map()` 操作符可以实现 `QueueSubscription` 接口，也可以有一个 `QueueSubscription` 成员，以便在上游也是 `QueueSubscription` 时做保存用。在 `requestFusion` 中，如果上游支持熔合，且下游不是线程边界，那就把请求发往上游，否则拒绝请求。

现在 `poll()` 不仅仅是转发给上游了，因为类型不同。不过我们有 mapper 函数。注意 null 表示的是终止事件或者暂无数据，所以 null 不应该被转化。

引入 `THREAD_BOUNDARY` 主要是因为 `map()`，或者更广的程度上来说，我们需要限制用户提供的代码所执行的线程。在操作符熔合中，mapper 在队列的出口线程被执行，可能是其他的线程。假设我们在 `observeOn` 之前有个耗时的操作需要在子线程执行。如果不启用熔合，那耗时计算的结果会被放到 `observeOn` 的队列中，然后在目标线程取出（主线程）。但是如果启用了熔合，目标线程将会执行 `poll()`，那就会在主线程执行耗时操作了。

`filter()` 也可以用同样的方式实现，但我们又得使用 `request(1)` 了：

~~~ java
static final class FilterSubscriber<T> implements Subscriber<T>, QueueSubscription<T> {
    // ...
 
    @Override
    public T poll() {
        for (;;) {
            T v = qs.poll();
 
            if (v == null || cancelled) {
                return null;
            }
 
            if (predicate.test(v)) {
                return v;
            }
 
            if (mode == ASYNC) {
                qs.request(1);
            }
        }
    }
 
    @Override
    public boolean isEmpty() {
        return qs.isEmpty();
    }
 
    // ...
}
~~~

由于 `filter()` 会丢弃数据，所以我们需要在 `poll()` 中循环到 `predicate` 通过测试，或者上游不再有数据。如果 `predicate` 检测不通过，在 `ASYNC` 模式下我们需要补充数据（注意，在同步模式下我们是不允许调用 `request()` 的）。

#### 实现支持熔合的消费者

通常来说操作符熔合对末端订阅者并不会太有用（也不会实现），例如 `Subscriber` 类或者 `subscribe(System.out::println)`。

这里我指的消费者也可以被看做中间操作符，但由于所有的操作符都可以看做订阅到上游的自定义 Subscriber，它们也就可以看做是消费者了。

我在前文提到，很多操作符都有头部队列（例如 `concatMap`，`observeOn`），或者消费内部 Publisher（例如 `flatMap`，`zip`）。它们是主要的消费者，也驱动着熔合的生命周期。

既然我们对 [`observeOn` 的实现原理](/AdvancedRxJava/2016/09/16/subscribeon-and-observeon/)已经比较熟悉了，那就来看看如何启用熔合：

~~~ java
static final class ObserveOnSubscriber<T> implements Subscriber<T>, Subscription {
 
    Queue<T> queue;
 
    int mode;
 
    Subscription s;
 
    // ...
 
    @Override
    public void onSubscribe(Subscription s) {
         this.s = s;
 
         if (s instanceof QueueSubscription) {
             QueueSubscription<T> qs = (QueueSubscription<T>)s;
 
             int m = qs.requestFusion(QueueSubscription.ANY
                  | QueueSubscription.THREAD_BOUNDARY);
 
             if (m == QueueSubscription.SYNC) {
                 q = qs;
                 mode = m;
                 done = true;
                  
                 actual.onSubscribe(this);
                  
                 return;
             }
 
             if (m == QueueSubscription.ASYNC) {
                 q = qs;
                 mode = m;
 
                 actual.onSubscribe(this);
 
                 s.request(prefetch);
 
                 return;
             }       
         }
 
         queue = new SpscArrayQueue<>(prefetch);
          
         actual.onSubscribe(this);
 
         s.request(prefetch);
    }
 
    @Override
    public void onNext(T t) {
        if (mode == QueueSubscription.NONE) {
            queue.offer(t);
        }
         
        drain();
    }
     
    void drain() {
 
        // ...
              
        if (mode != QueueSubscription.SYNC) {
            request(p);
        }
 
        // ...
 
    }
 
    // ...
 
}
~~~

启用熔合意味着两件事：1）`queue` 不能是 final 了，需要在 `onSubscribe` 中创建；2）`onNext` 中如果启用了熔合就不能把数据放进队列中。

熔合模式在 `onSubscribe` 中识别出上游是 `QueueSubscription` 之后请求。由于在 `drain()` 函数中的算法只能看到 `Queue` 接口，并不关心数据何时到来，所以我们请求 `ANY` 模式，以及或上 `THREAD_BOUNDARY` 表明我们是一个线程边界。这应该会阻止 `poll()` 函数中改变用户定义代码的顺序。

如果上游允许 `SYNC` 模式，我们就把 `QueueSubscription` 赋值给 `queue`，然后调用下游 `Subscriber` 的 `onSubscribe`。在这种模式下，我们遵守同步熔合的协议，并不预取 `prefetch` 个数据。`SYNC` 模式最大的好处就是如果 `poll()` 返回 null，就意味着数据流已经结束了。我们早已在标准的队列漏算法中利用了这一点：如果 `done` 被设置，且队列取出了 null，我们就结束了。但注意，我们还是需要略微调整队列漏算法，因为在 `SYNC` 模式下我们不能调用 `request` 函数。

如果上游允许 `ASYNC` 模式，我们也保存 `queue` 变量，但不能设置 `done`，因为我们不知道数据流何时终止，`poll()` 返回 null 只表示当前没有数据。此外，在通知了下游的 `Subscriber` 之后，我们还需要向上游预取数据，这样上游才知道触发它自己的数据源。

注意一旦 `requestFusion` 返回了 `SYNC` 或者 `ASYNC`，那就不能撤销了（你可以再次调用 `requestFusion()`，并换个参数，但这在当前是未定义的行为，这种行为将来可能被禁用），而且一旦数据按照这种模式已经开始分发了，那就绝对不可能撤销了。

### 微熔合的通用警告

根据我的经验，我的一些同事对微熔合表现得非常热情，他们希望在任何场景中启用。只要操作符有队列，他们就认为应该启用熔合。

我必须警告这种义无反顾的想法，因为启用熔合是需要条件的，而且通常都是消耗与收益的权衡：

+ 如果一个操作符时线程边界，就我目前的理解，我们不能同时熔合队列的头部和尾部。
+ 熔合又是会转移计算的时间，甚至空间（即便没有显式的线程边界）。
+ 操作符有队列不代表这个队列可以暴露或者被替换。`combineLatest` 是个好例子：就我的理解，队列中元素的后处理使得这个操作符无法启用尾部队列熔合。另一个例子是 `flatMap`，收集数据的逻辑我觉得不能集成到 `poll()`/`isEmpty()` 的尾部熔合中去。
+ 有些数据源（例如至多发出一个数据的数据源）可能不值得启用微熔合，宏熔合倒是更好的选择。
+ 熔合是额外的操作，实际上也可能会引入 bug，或者掩盖常规路径上的 bug（例如 `groupBy`），因此需要额外的关注。此外，它也增加了测试方法量，因为我们需要测试熔合和非熔合的版本（参见 `hide()`）。

为了鼓励大家一下，我有一个支持所有熔合的操作符例子：头部熔合和尾部熔合都支持，`flattenIterable`，或者 `concatMapIterable`/`flatMapIterable`。

## 总结

在本文中，我详细解说了操作符熔合的结构和协议，并且用例子展示了如何在数据源、中间操作符以及消费者环节启用熔合。

由于操作符熔合仍是一个活跃的研究领域，我不能说本文举出的是所有可能的熔合，而且我们也很期待听到社区分享一些有趣的可熔合调用链，或者是不能熔合的调用链。完整的熔合例子可以参见 [Rsc 仓库](https://github.com/reactor/reactive-streams-commons)。

此外，我希望这些熔合协议可以被纳入 **Reactive-Streams 2.0**，这样就能允许我们实现完整的、跨库的操作符熔合了。

下一个话题就要结束掉 `ConnectableObservable` 的系列了。
