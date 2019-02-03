---
layout: post
title: Operator 并发原语：串行访问（serialized access）（二），queue-drain
tags:
    - Operator
---

原文 [Operator concurrency primitives: serialized access (part 2)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_11.html){:target="_blank"}

## 介绍
在[Operator 并发原语：串行访问（serialized access）（一）](/AdvancedRxJava/2016/05/06/operator-concurrency-primitives/index.html){:target="_blank"}中，我介绍了串行访问的需求，它是为了让 RxJava 的一些方法以及操作符可以串行执行。我详细介绍了**发射者循环（emitter-loop）**，并展示了如何利用它来实现串行访问的约定。我想再次强调一点，**在大部分的单线程使用场景下，这种实现方式性能表现非常突出，因为 Java (JVM) 的 JIT 编译器会在检测到只有单线程使用时使用偏向锁和锁省略技术**。然而如果异步执行占了主要部分，或者发射操作在另一个线程执行，由于其阻塞（blocking）的原理，发射者循环就会出现性能瓶颈。

在本文中，我将介绍另一种非阻塞的串行实现方式，我称之为**队列漏（queue-drain）**。

## 队列漏（queue-drain）
相比于发射者循环，队列漏的代码相对简短：

~~~ java
class BasicQueueDrain {
    final AtomicInteger wip = new AtomicInteger();  // (1)
    public void drain() {
        // work preparation
        if (wip.getAndIncrement() == 0) {           // (2)
            do {
                // work draining
            } while (wip.decrementAndGet() != 0);   // (3)
        }
    }
}
~~~

它的实现原理是这样的：

1. 我们需要有一个可以进行原子自增操作的数字变量，我通常称之为 `wip` (work-in-progress 的缩写)。它用来记录需要被执行的任务数量，只要 Java runtime 底层实现具有支持（2）和（3）操作的原语，那我们实现的队列漏甚至是完全没有等待（阻塞）的（_译者注：CPU 如果支持 compare and set 指令，那自增操作就只需要一条指令_）。在 RxJava 中我们使用了 `AtomicInteger`，因为我们认为在通常的场景下，溢出是不可能发生的。当然，如果需要的话，它完全可以替换为 `AtomicLong`。
2. 我们利用原子操作，获取 `wip` 当前的值，并对它进行加一操作。只有把它从零增加到一的线程才能进入后面的循环，而其他所有的线程都将在加一操作之后返回。
3. 每当一件任务被取出（漏出）并处理完毕，我们就对 `wip` 进行减一操作。如果减到了零，那我们就退出循环。由于我们只有一个线程在进行减一操作，我们就能保证不会发生信号丢失。由于（2）和（3）操作都是原子性的，所以如果执行（3）时 `wip == 1`，而（2）先执行完毕，那（3）得到的 `wip` 值仍然为一，那就会再进行一次循环；而如果（3）先执行完毕，那（2）就是把 `wip` 从零增加到一的线程，那它就会进入后面的循环，而由于（3）先执行完毕，那它也就会退出循环了，两者没有任何干扰。

如果你还记得上篇中发射者循环的例子，我们就可以得出两种方式的数据结构之间有一些相似之处。

在发射者循环中，如果一个线程看到的是 `emitting == false` 而且把 `emitting` 置为了 `true`，那这个线程就获得了发射的权利，而在队列漏中，如果一个线程在原子自增操作中，是把 `wip` 从零增加到一，那这个线程就获得了漏的权利。在发射者循环中，如果一个线程没有获得发射的权利，那它会把 `missed` 置为 `true`，而在队列漏中，如果一个线程没有获得漏的权利，它就会继续增加 `wip`（增加为 2 或者更大）。在结束循环的检查中，发射者循环会检查是否 `missed` 已被置为 `true`，如果是就继续循环，在队列漏中，如果 `wip` 在减一之前大于一（减之后大于零），那就会继续循环。在发射者循环中，如果 `missed` 为 `false`，那就退出循环，在队列漏中，如果减一之后 `wip` 为零，就退出循环。

记住这些之后，我们就可以在上篇例子的基础上，利用队列漏方式实现一个非阻塞的 `ValueEmitterLoop` 版本了：

~~~ java
class ValueQueueDrain<T> {
    final Queue<T> queue = new MpscLinkedQueue<>();     // (1)
    final AtomicInteger wip = new AtomicInteger();
    Consumer consumer;                                  // (2)
 
    public void drain(T value) {
        queue.offer(Objects.requireNonNull(value));     // (3)
        if (wip.getAndIncrement() == 0) {
            do {
                T v = queue.poll();                     // (4)
                consumer.accept(v);                     // (5)
            } while (wip.decrementAndGet() != 0);       // (6)
        }
    }
}
~~~

和发射者循环的版本比较类似，但由于没有了 `synchronized` 代码块，队列漏更加精简：

1. 我们继续使用 [JCTools](https://github.com/JCTools/JCTools){:target="_blank"} 的优化版本来保存待消费的数据。如果可能的最大待消费数据数量是确定的，那就可以使用 `MpscArrayQueue`，不过需要注意，它提供的 API 与 `offer` / `poll` 相比，[并不是非阻塞的](http://psy-lob-saw.blogspot.com/2015/04/porting-dvyukov-mpsc.html){:target="_blank"}。
2. 我们把队列里的数据一个一个漏出来给 `consumer`。
3. 首先，我们把不为 `null` 的数据加入到队列中。
4. 获得漏的权利的线程负责从队列中取出数据。由于 `wip` _大体上_ 记录了队列中的数据个数，`poll()` 不会返回 `null`（_`wip` 的值一定不小于 `queue` 的元素个数_）。
5. 我们把取出的数据传递给 `consumer`。
6. 如果 `wip` 减一之后不为零，我们就继续循环，否则退出循环。注意，确实有可能执行到（6）的时候，队列中还有元素，但这并不是问题，因为即便（3）的调用和增加 `wip` 之间存在延迟，当前的循环也可以退出，因为紧接着增加 `wip` 的时候，又会有新的线程进入循环。

如果我们在单线程场景下对 `ValueEmitterLoop` 和 `ValueQueueDrain` 进行性能测试，在启动完毕（JIT 生效）之后，后者的吞吐量会更低。

出现这样的现象，是因为队列漏方式存在无法避免的原子操作开销，即便在没有多线程进程的场景下，也会多消耗几个 CPU 周期，这是由现代多核 CPU 强制缓冲区写刷新（mandatory write-buffer flush）导致的。我们每次 `drain()` 一个数据的时候，有两次增减原子操作，以及 `MpscLinkedQueue` 使用的原子操作。而一旦 `ValueListEmitterLoop` 完成了锁优化之后，性能就会更好。

_注意，我在上文提到的 benchmark 在 `consumer` 的回调中执行的代码非常简单，以此来测量串行执行的开销。不同的 `consumer` 以及不同的调用分布，都可能会影响测试结果。_

我们可以优化一下我们的队列漏实现，让它在没有并发的时候，也能利用锁优化技术。

但是需要注意的是，这样的优化会让代码变得更长更复杂。所以我强烈建议**先对非优化版本进行测量，或者尝试 set-to-1 trick**（我将在后面讲解）。

下面我们看看第一个优化版本：

~~~ java
class ValueQueueDrainFastpath<T> {
    final Queue<T> queue = new MpscLinkedQueue<>();
    final AtomicInteger wip = new AtomicInteger();
    Consumer consumer;
 
    public void drain(T value) {
        Objects.requireNonNull(value);
        if (wip.compareAndSet(0, 1)) {          // (1)
            consumer.accept(value);             // (2)
            if (wip.decrementAndGet() == 0) {   // (3)
                return;
            }
        } else {
            queue.offer(value);                 // (4)
            if (wip.getAndIncrement() != 0) {   // (5)
                return;
            }
        }
        do {
            T v = queue.poll();                 // (6)
            consumer.accept(v);
        } while (wip.decrementAndGet() != 0);
    }
}
~~~

尽管 `ValueQueueDrainFastpath` 看上去使用了更多的原子操作，但实际上在单线程场景下，它不会调用到 `offer` 和 `poll`，因此性能更好一些。它的工作原理如下：

1. 在确保数据不为 `null` 之后，我们尝试对 `wip` 执行 CAS （compare-and-swap）操作，如果旧的值为 0，我们就将其置为 1（尝试把 `wip` 从 0 变为 1）。（注意，在 CAS 操作之前加上 `wip.get() == 0 &&` 判断也许可以提升性能，但这种提升是严重依赖于 CPU 类型以及调用分布的。我在不同的 CPU 类型下运行 benchmark，没有固定的结论。）
2. （_如果我们成功把 `wip` 从 0 变为 1_）我们直接把数据传递给 `consumer`，不经过 `queue` 中转。这一点就是本优化版本的性能提升点。
3. 调用 `consumer` 回调之后，我们需要对 `wip` 执行减一操作，而如果没有其他的线程调用 `drain()` 并且执行到（5），那我们就可以退出了。有人可能会觉得在这里我们也可以进行 CAS 操作（从 1 到 0），但如果 CAS 失败，我们仍然需要对 `wip` 进行减一操作，然后再进入到后面的循环中。
4. 如果（1）处的 CAS 失败，我们就进入到常规的处理流程（慢路径，slow-path），先把数据加入到队列中。
5. 把数据加入到队列之后，我们把 `wip` 加一，如果加一之前 `wip` 不为零，我们就返回了（我们返回的条件必须是非零，因为快路径和慢路径共用了同一个漏循环）。这个检查是非常必要的，因为有可能在当前线程执行到（5）之前，就有另一个线程进入了快路径并且执行完毕返回了，那此时 `wip` 依然是 0，所以当前线程依然是把 `wip` 从 0 变为 1 的线程，依然应该进入漏循环中。
6. 无论是快路径还是慢路径进入了漏循环，我们的执行就和普通队列漏循环一样了。当有线程在执行漏循环时，其他线程都讲在（1）处的 CAS 失败，进而进入到慢路径中。

需要注意的是，这个快路径优化版本是一个折衷方案（tradeoff）：快路径优化（没有额外的队列原子操作）与慢路径恶化（多了一次失败的 CAS 操作）的一个折衷（或者加一个 `wip.get() == 0` 判断，能省去慢路径的 CAS 失败，但却取决于调用分布）。所以我要再次强调，**先测量，别假设**。

观察仔细的朋友可能已经注意到，`ValueQueueDrainFastpath` 和上篇中的 `ValueListEmitterLoop` 很相似：获得发射权利（漏权利）的线程，都先直接把数据传给 `consumer`，跳过队列操作，这是快路径；而如果此时有其他线程在循环中了（此时 `emitting` 为 `true` 或者 `wip` 大于 0），那就回退到慢路径中。

除了上述的快路径优化，我们还有一种优化方案可以减少队列漏中的原子操作。还记得我们在前文中对发射者循环和队列漏的对比吗？只要 `wip` 值大于 1，就意味着还有更多的数据，而队列本身可以告诉我们数据是否被处理完了。所以如果我们每次都把队列漏完，并且（_漏完之后_）在我们对 `wip` 进行减一操作之前如果又有新的数据被加入到队列中，我们就继续循环，那我们就可以在每次漏一个数据的时候减少一次原子操作：

~~~ java
class ValueQueueDrainOptimized<T> {
    final Queue<T> queue = new MpscLinkedQueue<>();
    final AtomicInteger wip = new AtomicInteger();
    Consumer consumer;

    public void drain(T value) {
        queue.offer(Objects.requireNonNull(value));
        if (wip.getAndIncrement() == 0) {
            do {
                wip.set(1);                              // (1)
                T v;
                while ((v = queue.poll()) != null) {     // (2)
                    consumer.accept(v);
                }
            } while (wip.decrementAndGet() != 0);        // (3)
        }
    }
}  
~~~

`ValueQueueDrainOptimized` 和 `ValueQueueDrain` 看起来很相似，但工作原理略有不同：

1. 一旦我们进入漏循环，我们就把 `wip` 置为 1。由于我们会一次性把队列漏完，_如果执行（3）操作时，没有其他线程干扰_，那我们就无需再次循环了，因为队列已经漏空了，所以这里我们可以直接把 `wip` 置为 1。这和上篇的 `EmitterLoopSerializer` 中，我们在重新循环之前把 `missed` 置为 `false` 道理是一样的。
2. 接下来我们就通过一个内循环一次性漏空整个队列，而不是逐个进行。我们不需要判断 `queue.size()` 或者 `queue.isEmpty()`，因为如果 `poll` 返回了 `null`，就表明队列空了。如此我们就可以在每次漏一个数据的时候，节约一次原子操作。
3. 内循环退出后，我们对 `wip` 减一，如果减成零了，那我们就退出循环。在一种理想情况下，如果在执行（1）之前爆发性地往队列里添加了大量数据，我们会一次性把它们都处理完，并且由于我们把 `wip` 置为了 1，我们的漏循环只会执行一次。如果在执行（3）之前又有数据被加入到了队列中，这不会出现问题，因为原子操作会保证数据同步：所以要么（3）操作没有把 `wip` 减到零，进而重新循环，要么（3）把 `wip` 减到零并退出，进而让 `if (wip.getAndIncrement() == 0)` 通过并进入循环。

需要注意的是，这一优化版本同样有其限制。尽管我们把后续的所有数据一次性批量漏空，（2）处的缓存一致性（cache-coherence）保证，可能会让我们在某些情况下性能没有任何提升，所以这一版本的性能会取决于对 `drain()` 的调用分布。

通过上面的例子和解释，我们可能会觉得队列漏问题多多：它的优化空间以及性能表现都严重依赖于调用分布，因此通常情况下都会倾向于使用发射者循环。

但是，有一个几乎任何人都会使用的操作符就是使用的队列漏：`observeOn`。它目前的实现，使用了 `ValueQueueDrainOptimized` 中介绍的优化方案，因为它对队列（`SpscArrayQueue`）的 `offer` 和 `poll` 的操作都只在单一的线程中（两者是不同的线程）（当然，特定的 backpressure 场景除外，我将在后续的文章中进行更多的分析）。

## 总结
在本文中，我介绍了实现串行访问的第二种方案，我称之为**队列漏（queue-drain）**。我展示了一个基础的实现版本和两种优化的版本，优化版本的性能都依赖于对 `drain()` 的调用分布（以及 CPU 类型）。所以请**务必对你的实现版本进行测量**。

它使用的非阻塞无锁（甚至无等待）数据结构使得它在中高程度并发场景下性能更优，以及确定添加数据和消耗数据分别发生在不同的线程的场景下性能更优。

在接下来的文章中，我将介绍多种线程安全的 `Producer` 实现，它们其中一些就利用了**发射者循环（emitter-loop）**和**队列漏（queue-drain）**，我将在后续的文章中详细介绍。
