---
layout: post
title: FlatMap 技术揭秘（二）
tags:
    - Operator
---

原文 [FlatMap (part 2)](http://akarnokd.blogspot.com/2016/03/flatmap-part-2.html){:target="_blank"}。

## 介绍

在本文中，我们将会丰富我们 flatMap 实现的功能，并提升它的性能。

RxJava 的 flatMap 提供了最大并发数限制，也就是最多同时允许订阅到 mapper 产生的 Observable 的数量，还提供了错误延迟功能，允许延迟任何上游产生的错误，包括主上游。

## 并发限制

由于历史原因，RxJava 的 flatMap（以及我们在[上篇](/AdvancedRxJava/2017/04/10/flatmap-part-1/index.html)中的实现）对主上游来说都是处于无尽模式的。这对不太频繁的主上游，以及持续较短的内部 Observable 也许是可行的。然而即便主上游可以按照特定的频率发射，例如 range()，但内部的 Observable 也可能会占用有限的资源，例如网络连接。

那么问题在于，我们如何确保只有用户指定数量的 Observable 可以被同时订阅？我们怎么确保有些主上游只发射出指定数量的数据？

答案当然是 **backpressure**。

为了限制 flatMap 的并发，主要思想是先向主上游请求 `maxConcurrency`，再在每次有内部 Observable 终止时调用 `request(1)`。

让我们改变 OpFlatMap 和 FlatMapSubscriber 的实现，加入这个 maxConcurrency 参数：

~~~ java
final int maxConcurrency;
 
public OpFlatMap(Func1<? super T, ? extends Observable<? extends R>> mapper,
        int prefetch, int maxConcurrency) {
    this.mapper = mapper;
    this.prefetch = prefetch;
    this.maxConcurrency = maxConcurrency;
}
  
@Override
public Subscriber<T> call(Subscriber<? super R> t) {
    FlatMapSubscriber<T, R> parent = 
        new FlatMapSubscriber<>(t, mapper, prefetch, maxConcurrency);
    parent.init();
    return parent;
}
~~~

我们约定，把 `Integer.MAX_VALUE` 作为使用原来无尽模式的标志：

~~~ java
final int maxConcurrency;
 
public FlatMapSubscriber(Subscriber<? super R> actual,
        Func1<? super T, ? extends Observable<? extends R>> mapper,
        int prefetch, int maxConcurrency) {
    this.actual = actual;
    this.mapper = mapper;
    this.prefetch = prefetch;
    this.csub = new CompositeSubscription();
    this.wip = new AtomicInteger();
    this.requested = new AtomicLong();
    this.queue = new ConcurrentLinkedQueue<>();
    this.active = new AtomicInteger(1);
    this.error = new AtomicReference<>();
 
    this.maxConcurrency = maxConcurrency;
    if (maxConcurrency != Integer.MAX_VALUE) {
        request(maxConcurrency);
    }
}
~~~

最后，我们改变 innerComplete 的实现，向主上游请求一个数据：

~~~ java
void innerComplete(Subscriber<?> inner) {
    csub.remove(inner);
 
    request(1);
 
    onCompleted();
}
~~~

这只是对 backpressure 很直观的应用。但要注意 innerComplete 可能被内部 Observable 触发并发调用，所以主上游的请求处理逻辑必须是[线程安全和可重入的](/AdvancedRxJava/2016/05/18/operator-concurrency-primitives-3/index.html)。

## 错误延迟

很多标准的操作符都会默认在收到 onError 之后提前结束事件流。但如果操作符涉及到多个上游，我们有时也会希望先处理完正常事件，最后再处理可能的错误。

~~~ java
final boolean delayErrors;
 
public OpFlatMap(Func1<? super T, ? extends Observable<? extends R>> mapper,
        int prefetch, int maxConcurrency, boolean delayErrors) {
    this.mapper = mapper;
    this.prefetch = prefetch;
    this.maxConcurrency = maxConcurrency;
    this.delayErrors = delayErrors;
}
  
@Override
public Subscriber<T> call(Subscriber<? super R> t) {
    FlatMapSubscriber<T, R> parent = 
        new FlatMapSubscriber<>(t, mapper, prefetch, maxConcurrency, delayErrors);
    parent.init();
    return parent;
}
 
// ...
 
    final boolean delayErrors;
 
    public FlatMapSubscriber(Subscriber<? super R> actual,
            Func1<? super T, ? extends Observable<? extends R>> mapper,
            int prefetch, int maxConcurrency, boolean delayErrors) {
        this.actual = actual;
        this.mapper = mapper;
        this.prefetch = prefetch;
        this.csub = new CompositeSubscription();
        this.wip = new AtomicInteger();
        this.requested = new AtomicLong();
        this.queue = new ConcurrentLinkedQueue<>();
        this.active = new AtomicInteger(1);
        this.error = new AtomicReference<>();
         
        this.maxConcurrency = maxConcurrency;
        if (maxConcurrency != Integer.MAX_VALUE) {
            request(maxConcurrency);
        }
         
        this.delayErrors = delayErrors;
    }
~~~

flatMap 的延迟错误处理，延迟部分倒是很简单，但和错误相关的处理相对复杂一些：最后我们只能发出一个 onError 事件，不管此前发生了多少个错误（主上游或者内部上游）。显然，在结束之前都把第一个错误保存起来是一种办法，但把其他的错误丢掉可能并不是我们希望的效果。解决办法就是把所有的 Throwable 保存在某种数据结构中，最后发出一个 CompositeException。

使用支持并发的 `Queue<Throwable>` 是选择之一，这也是 RxJava 的做法。但我们也可以复用已有的 AtomicReference，并通过 CAS 来积攒错误：

~~~ java
@Override
public void onError(Throwable e) {
    if (delayErrors) {
        for (;;) {
            Throwable current = error.get();
            Throwable next;
            if (current == null) {
                next = e;
            } else {
                List<Throwable> list = new ArrayList<>();
                if (current instanceof CompositeException) {
                    list.addAll(((CompositeException)current).getExceptions());
                } else {
                    list.add(current);
                }
                list.add(e);
                 
                next = new CompositeException(list);
            }
             
            if (error.compareAndSet(current, next)) {
                if (active.decrementAndGet() == 0) {
                    drain();
                }                        
                return;
            }
        }
    } else {
        if (error.compareAndSet(null, e)) {
            unsubscribe();
            drain();
        } else {
            RxJavaPlugins.getInstance()
            .getErrorHandler().handleError(e);
        }
    }
}
~~~

在循环中，我们取出此前的错误，如果它是 null，我们就把它置为新的异常。如果早已发生过错误，那我们就创建一个 CompositeException，用来容纳此前的异常和新的异常。但如果此前的错误就已经是 CompositeException 类型了，我们就把此前异常容器里的所有异常展开，这让下游最终收到 onError 时看到的异常比较简单，只有一层 CompositeException。由于我们现在把 onError 和 onCompleted 都作为非全局的终止事件了，我们需要在 onError 中递减 active 计数器，在递减到 0 时调用 drain。

考虑到 Java 7 的 `Throwable.addSuppressed`，有人可能会用它来收集错误，但它有一些缺点：它使用了 `synchronized`，而且需要提前创建一个异常容器对象（一是需要耗费一定的时间，二是即便没有错误也需要创建这个容器对象）。此外，修改已有的异常也是比较令人费解的一件事。

由于 innerError 不再是立即终止的，我们需要修改它的逻辑，把内部的 subscriber 移除，并在运行在有限并发模式下时向主上游请求新的数据：

~~~ java
void innerError(Throwable ex, Subscriber<?> inner) {
    if (delayErrors) {
        csub.remove(inner);
        request(1);
    }
    onError(ex);
}
~~~

最后，我们需要调整 drain() 函数。上篇的实现中我们检测到错误之后，立即就发往了下游。现在要改成只有当共享队列中所有的数据都发送完毕之后再发出错误（就像 onCompleted 一样）：

~~~ java
boolean done = active.get() == 0;
if (!delayErrors) {
    Throwable ex = error.get();
    if (ex != null) {
        actual.onError(ex);
        return;
    }
}
  
Object o = queue.poll();
  
if (done && o == null) {
    Throwable ex = error.get();
    if (ex != null) {
        actual.onError(ex);
    } else {
        actual.onCompleted();
    }
    return;
}
  
if (o == null) {
    break;
}
~~~

原来的错误发射的逻辑在判断 delayErrors 为 false 之后。否则我们就把错误的检查放在了所有的上游都结束，且队列清空之后。如有错误，我们就发出 onError，否则发出 onCompleted。

此外，我们还需要更新 `e == r` 的处理逻辑（这种情况下我们发出了被请求的数量，那么下一个就要是终止事件了）：

~~~ java
if (e == r) {
    if (actual.isUnsubscribed()) {
        return;
    }
    boolean done = active.get() == 0;
     
    if (!delayErrors) {
        Throwable ex = error.get();
        if (ex != null) {
            actual.onError(ex);
            return;
        }
    }
      
    if (done && queue.isEmpty()) {
        Throwable ex = error.get();
        if (ex != null) {
            actual.onError(ex);
        } else {
            actual.onCompleted();
        }
        return;
    }
}
~~~

基本和上面一样，但这里我们是检查 `isEmpty()` 而不是 `poll()` 的返回值，因为如果队列不为空，我们不希望消费这个数据。

现在我们完成了 OpFlatMap 的功能扩展（当然别忘了把 `FlatMapInnerSubscriber.onError` 的实现改为 `parent.innerError(e, this);`）。

## 优化队列的性能

队列旁路的优化有其限制，当所有的上游发射速度都很快时，一直都存在竞争，因此几乎不会被触发。

竞争会影响到共享队列以及 wip 计数器，因此我们可以通过避免这两个竞争点来提升一部分性能。然而 wip 计数器无法避免，所以让我们看看队列的优化。

问题在于所有的上游都共用一个队列，所以会在 offer() 调用处发生竞争，因此需要一个多生产者的队列，而其内部使用了重量级的 `getAndSet()` 或者 `getAndIncrement()` 原子操作。

然而，由于每个上游自身都是串行的，我们的生产者实际都可以看做是单线程的，最多会有 N 路并发，而由于漏循环的存在，我们只会有一个消费者。

解决办法就是为每个上游都准备一个单独的单生产者、单消费者的队列，然后在漏循环中，从每个队列中收集数据。这给了 [JCTools 的高性能 SpscArrayQueue](https://github.com/JCTools/JCTools/blob/master/jctools-core/src/main/java/org/jctools/queues/SpscArrayQueue.java) 一个绝佳的机会。我们还可以使用数组的实现版本，因为我们的 prefetch 值预期是比较小的，RxJava 2.x 默认值是 128。

这需要对 FlatMapInnerSubscriber 和 FlatMapSubscriber 做一些修改：

~~~ java
static final class FlatMapInnerSubscriber<T, R> extends Subscriber<R> {
    final FlatMapSubscriber<T, R> parent;
  
    final int prefetch;
     
    volatile Queue<Object> queue;
 
    volatile boolean done;
     
    public FlatMapInnerSubscriber(
            FlatMapSubscriber<T, R> parent, int prefetch) {
        this.parent = parent;
        this.prefetch = prefetch;
        request(prefetch);
    }
     
    @Override
    public void onNext(R t) {
        parent.innerNext(this, t);
    }
      
    @Override
    public void onError(Throwable e) {
        done = true;
        parent.innerError(e, this);
    }
      
    @Override
    public void onCompleted() {
        done = true;
        parent.innerComplete(this);
    }
  
    void requestMore(long n) {
        request(n);
    }
     
    Queue<Object> getOrCreateQueue() {
        Queue<Object> q = queue;
        if (q == null) {
            q = new SpscArrayQueue<>(prefetch);
            queue = q;
        }
        return q;
    }
}
~~~

FlatMapInnerSubscriber 加了两个成员，一是 prefetch，用于后面创建 SpscArrayQueue 对象，二是 Queue 对象。此外，我们还需要知道上游是否已经停止，这通过 done 成员来实现。当然，我们也可以提前创建队列，但这就会浪费前面提到的快速路径的收益了，如果快速路径成功生效，那我们就不需要队列了。如果我们终究需要队列，getOrCreateQueue 函数将会创建队列。注意，如果最终还是需要队列，它将被单一的线程创建，但会被漏循环中的线程访问，因此需要用 volatile 修饰符。

接下来就是修改 `innerNext()` 让它可以使用每个上游单独的队列，而不是共享队列：

~~~ java
void innerNext(FlatMapInnerSubscriber<T, R> inner, R value) {
    Object v = NotificationLite.instance().next(value);
     
    if (wip.get() == 0 && wip.compareAndSet(0, 1)) {
        if (requested.get() != 0L) {
            actual.onNext(value);
            BackpressureUtils.produced(requested, 1);
            inner.requestMore(1);
        } else {
            Queue<Object> q = inner.getOrCreateQueue();

            q.offer(v);
        }
         
        if (wip.decrementAndGet() != 0) {
            drainLoop();
        }
        return;
    }
     
    Queue<Object> q = inner.getOrCreateQueue();
     
    q.offer(v);
    drain();
}
~~~

改变只在于：如果出现了竞争，或者下游没有发出请求，就把原来的共享队列替换为 `inner.getOrCreateQueue()`（这里我们已经可以把共享队列从 FlatMapSubscriber 中移除了，但我们暂且留着）。

不幸的是，每个上游独立队列的方式给我们带来了一些麻烦，因为 `drainLoop()` 不能使用共享队列，而我们又需要知道当前正在活跃的上游，但是 CompositeSubscription 并没有把它的内容暴露出来。此外，CompositeSubscription 内部使用了 HashSet，它需要保证能线程安全地遍历，为大部分情况增加了如此多的开销会让我们其他的努力付诸东流。

这里我们可以套用一下以前我们在 Subject 和 ConnectableObservable 中使用的 [copy-on-write 模式的 Subscriber 管理技术](/AdvancedRxJava/2016/07/29/operator-concurrency-primitives-subscription-containers-3/index.html)。这让我们有了一个很漂亮的 FlatMapInnerSubscriber 数组，并可以摆脱 csub 和 active 成员。

~~~ java
@SuppressWarnings("rawtypes")
static final FlatMapInnerSubscriber[] EMPTY = new FlatMapInnerSubscriber[0];
@SuppressWarnings("rawtypes")
static final FlatMapInnerSubscriber[] TERMINATED = new FlatMapInnerSubscriber[0];
 
final AtomicReference<FlatMapInnerSubscriber<T, R>[]> subscribers;
 
volatile boolean done;
 
@SuppressWarnings("unchecked")
public FlatMapSubscriber(Subscriber<? super R> actual,
        Func1<? super T, ? extends Observable<? extends R>> mapper,
        int prefetch, int maxConcurrency, boolean delayErrors) {
    this.actual = actual;
    this.mapper = mapper;
    this.prefetch = prefetch;
    this.wip = new AtomicInteger();
    this.requested = new AtomicLong();
    this.error = new AtomicReference<>();
     
    this.subscribers = new AtomicReference<>(EMPTY);
     
    this.maxConcurrency = maxConcurrency;
    if (maxConcurrency != Integer.MAX_VALUE) {
        request(maxConcurrency);
    }
    this.delayErrors = delayErrors;
}
~~~

我们有标记空状态和终止状态的标记数组，以及一个 `volatile done` 成员，当主上游终止后它会被置为 true。初始化的逻辑也需要改变了，此外我们还需要常规的 `add()`，`remove()` 和 `terminate()` 函数：

~~~ java
public void init() {
    add(Subscriptions.create(this::terminate));
    actual.add(this);
    actual.setProducer(new Producer() {
        @Override
        public void request(long n) {
            childRequested(n);
        }
    });
}
 
@SuppressWarnings("unchecked")
void terminate() {
    FlatMapInnerSubscriber<T, R>[] a = subscribers.get();
    if (a != TERMINATED) {
        a = subscribers.getAndSet(TERMINATED);
        if (a != TERMINATED) {
            for (FlatMapInnerSubscriber<T, R> inner : a) {
                inner.unsubscribe();
            }
        }
    }
}
 
boolean add(FlatMapInnerSubscriber<T, R> inner) {
    for (;;) {
        FlatMapInnerSubscriber<T, R>[] a = subscribers.get();
        if (a == TERMINATED) {
            return false;
        }
        int n = a.length;
        @SuppressWarnings("unchecked")
        FlatMapInnerSubscriber<T, R>[] b = new FlatMapInnerSubscriber[n + 1];
        System.arraycopy(a, 0, b, 0, n);
        b[n] = inner;
        if (subscribers.compareAndSet(a, b)) {
            return true;
        }
    }
}
 
@SuppressWarnings("unchecked")
void remove(FlatMapInnerSubscriber<T, R> inner) {
    for (;;) {
        FlatMapInnerSubscriber<T, R>[] a = subscribers.get();
        if (a == TERMINATED || a == EMPTY) {
            return;
        }
        int n = a.length;
        int j = -1;
        for (int i = 0; i < n; i++) {
            if (a[i] == inner) {
                j = i;
                break;
            }
        }
         
        if (j < 0) {
            return;
        }
         
        FlatMapInnerSubscriber<T, R>[] b;
        if (n == 1) {
            b = EMPTY;
        } else {
            b = new FlatMapInnerSubscriber[n - 1];
            System.arraycopy(a, 0, b, 0, j);
            System.arraycopy(a, j + 1, b, j, n - j - 1);
        }
        if (subscribers.compareAndSet(a, b)) {
            return;
        }
    }
}
~~~

onNext 函数有一个小变化，我们的订阅调用需要增加一个判断条件，以免操作符已经被取消订阅：

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
      
    FlatMapInnerSubscriber<T, R> inner = 
            new FlatMapInnerSubscriber<>(this, prefetch);
 
    if (add(inner)) {
        o.subscribe(inner);
    }
}
~~~

onError 函数也需要一个小变化，由于这里没有 active 计数器了，所以我们一定调用 drain 函数：

~~~ java
if (error.compareAndSet(current, next)) {
    drain();
    return;
}
~~~

onCompleted 也不需要递减 active 计数器了，但它需要设置 done 标记：

~~~ java
@Override
public void onCompleted() {
    done = true;
    drain();
}
~~~

innerError 和 innerCompleted 也变简单了：

~~~ java
void innerError(Throwable ex, FlatMapInnerSubscriber<T, R> inner) {
    onError(ex);
}
  
void innerComplete(FlatMapInnerSubscriber<T, R> inner) {
    drain();
}
~~~

当然，所有的简化都一如既往地把复杂度转移到了其他的地方。这里我们的漏循环变得更复杂了：我们需要遍历所有的上游，漏出它们的队列，并请求新数据，包括向主上游发出请求。

~~~ java
void drainLoop() {
      
    int missed = 1;
      
    for (;;) {
        boolean d = done;
         
        FlatMapInnerSubscriber<T, R>[] a = subscribers.get();
         
        long r = requested.get();
        long e = 0L;
        int requestMain = 0;
        boolean again = false;
         
        if (isUnsubscribed()) {
            return;
        }
~~~

漏循环现在多了一些局部变量。我们提前获取 Subscriber 数组，并引入了一个发往主上游的请求计数器，以及一个标记外层循环需要继续的标记变量。注意，我们需要在获取 Subscriber 数组之前获取 done 标记，这样能避免和 onNext 的竞争。

~~~ java
if (!delayErrors) {
    Throwable ex = error.get();
    if (ex != null) {
        actual.onError(ex);
        return;
    }
}
 
if (d && a.length == 0) {
    Throwable ex = error.get();
    if (ex != null) {
        actual.onError(ex);
    } else {
        actual.onCompleted();
    }
    return;
}
~~~

接下来我们处理了延迟的错误，以及非延迟的错误。注意我们这里并没有单独使用 done 标记，而是结合了内部 Subscriber 数组的长度，只有主上游终止且没有活跃的内部 Subscriber（空数组）后，我们才算终止。

~~~ java
for (FlatMapInnerSubscriber<T, R> inner : a) {
    if (isUnsubscribed()) {
        return;
    }
     
    d = inner.done;
    Queue<Object> q = inner.queue;
    if (q == null) {
        if (d) {
            remove(inner);
            requestMain++;
            again = true;
        }
    } else {
~~~

接下来我们遍历 Subscriber 数组，检查它们的队列中是否有数据（只要它确实创建了队列）；有可能快速路径生效了，因此这个上游并没有通过 `getOrCreateQueue()` 创建过队列。这时我们只需要在它终止后，从数组中移除，并递增向主上游的请求计数。

~~~ java
long f = 0L;
 
while (e != r) {
    if (isUnsubscribed()) {
        return;
    }
     
    d = inner.done;
    Object v = q.poll();
    boolean empty = v == null;
     
    if (d && empty) {
        remove(inner);
        requestMain++;
        again = true;
    }
     
    if (empty) {
        break;
    }
     
    actual.onNext(NotificationLite.<R>instance().getValue(v));
     
    e++;
    f++;
}
 
if (f != 0L) {
    inner.requestMore(f);
}
 
if (e == r) {
    if (inner.done && q.isEmpty()) {
        remove(inner);
        requestMain++;
        again = true;
    }
    break;
}
~~~

这就是一个寻常的漏循环了，只是增加了移除 Subscriber、补充数据的逻辑，以及在发射数量达到请求数量时推出循环。注意 f 计数器用来统计被 FlatMapInnerSubscriber 消费的数据量。

~~~ java
                }
            }
 
            if (e != 0L) {
                BackpressureUtils.produced(requested, e);
            }
            if (requestMain != 0) {
                request(requestMain);
            }
             
            if (again) {
                continue;
            }
             
            missed = wip.addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        }
    }
}
~~~

最后的部分就是请求计数更新，补充数据，以及错过的调用检查逻辑了。

## 内部请求打包

在结束本文之前，让我们对最新的 flatMap 的结构进行最后一个小优化。

如果我们仔细看 innerNext() 的代码就会发现，无论快速路径何时发生，我们都是只请求一个数据进行补充。假设主上游是 `range()`，这样逐个请求的操作，会在每次发出数据之后，都带来一次原子递增操作，而这将会带来更多开销。

幸运的是，内部上游的 prefetch 值是固定的，因此我们可以定义一个重新请求的阈值，把这些单一的请求进行打包，以减少请求管理的开销。

这个阈值可以是 1~prefetch 之间的任意值，而且通常这个值取决于上游发射数据的模式。上游可能在任意阈值上都能发挥得更好。不幸的是，库里面无法为每个上游设置不同的阈值，而任何自适应的逻辑都可能带来过多开销，使得这一优化反而起反作用。因此 RxJava 使用的是 `prefetch / 2`（最近我尝试使用 `75% * prefetch`）。

这一优化方案需要为 FlatMapInnerSubscriber 增加两个成员，以及修改其 requestMore() 函数：

~~~ java
final int limit;
 
long produced;
 
public FlatMapInnerSubscriber(
        FlatMapSubscriber<T, R> parent, int prefetch) {
    this.parent = parent;
    this.prefetch = prefetch;
 
    this.limit = prefetch - (prefetch >> 2);
 
    request(prefetch);
}
 
void requestMore(long n) {
    long p = produced + n;
    if (p >= limit) {
        produced = 0;
        request(p);
    } else {
        produced = p;
    }
}
~~~

## 总结

在本文中，我展示了如何优化 flatMap 操作符的功能以及性能。勤奋的读者可能会检查我们是否达到了 RxJava 实际的实现，但答案是：还没有。为了发布这样一篇已经很长了的博文，我不得不去掉了其他几个我们可以应用的优化。

首先，最后一个内部上游可能会是新的下游请求到来时，将要恢复发出数据的对象，我们可以利用这种可能性。把 FlatMapInnerSubscriber 数组的索引保存起来，可以帮助我们实现这一优化。

第二个优化就是所谓的**标量优化（scalar-optimization）**了，它可以优化我们 flatMap 的目标 Observable 是 `Observable.just()` 的情况，可以避免订阅到这些 Observable 的开销。这一优化为 `drainLoop()` 函数增加了大量的逻辑，而且还需要一个单独的队列旁路逻辑。

在本系列的下一篇中，我将实现这两个优化，以及其他一些更好的优化。但是为了理解这些神秘的优化，包括标量优化，我们必须先学习一些新的知识，而这不仅要求我们对 flatMap 的内部逻辑有十分深刻的理解，还要求我们对其他操作符的理解也要十分深刻。

我们称之为**操作符熔合（operator fusion）**。
