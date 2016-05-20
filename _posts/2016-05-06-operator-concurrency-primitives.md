---
layout: post
title: Operator 并发原语：串行访问（serialized access）（一）
tags:
    - Operator
---

原文 [Operator concurrency primitives: serialized access (part 1)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives.html){:target="_blank"}

## 介绍
[RxJava 库](https://github.com/ReactiveX/RxJava){:target="_blank"}中最重要的要求就是 `Observer/Subscriber` 的 `onNext`，`onError` 以及 `onCompleted` 方法需要是串行调用的（_译者注：一个事件流，它的事件当然需要是串行发生的，不可能同一个流的多个事件同时发生_）。尽管我们称之为 **serialized access**，但它和 Java 的 `Serializable` 接口或者数据序列化没有任何关系。

这样的串行访问控制在遇到并发情况时将发挥重要作用。一个典型的例子是使用者利用 `merge`，`zip` 或者 `combineLatest` 操作符把多个数据流合并为一个数据流。

当然也存在一些并非显而易见的场景，例如 `takeUntil` 操作符。它需要两个 `Observable` 参数，在第二个 `Observable` 的发射（emission）或者终止事件之前，我们会转发第一个 `Observable` 的事件，而在第二个 `Observable` 的发射或者终止事件之后，经过 `takeUntil` 操作符之后的事件流就会被终止了。由于作为参数的两个事件流都是可以异步执行的，所以它们都可以随时发出 `onXXX` 事件，但是我们仍需要保证经过 `takeUntil` 操作符之后的事件是串行发生的。

实现这样的串行访问要求，最典型的方式就是为 `Observer` 的方法加上 `synchronized` 关键字。这种方式可以达到预期的效果，但容易出现死锁。如果下游的（downstream）操作符在和上游的（upstream）操作符进行互动时执行了锁操作，就会导致死锁（_译者：这句话没太懂，难道是下游操作符等待上游的锁，而上游的操作符也在等下游的锁？也许需要看看源码才能理解_）。这样的死锁同样可能出现在其他 RxJava 的基础类型中。

所以，在 RxJava 中我们禁止在持有锁的时候调用以下任何方法：

+ `Observer`: `onNext`, `onError`, `onCompleted`
+ `Subscriber`: `onStart`, `setProducer`, `request`
+ `Subscription`: `unsubscribe`
+ `Scheduler.Worker`: `schedule`, `schedulePeriodically`
+ `Producer`: `request`

（不幸的是，我们无法自动检查这一点，所以我们在处理 pr 时会人工检查）（_译者：也许可以编写自定义的 lint rule？_）

由于锁已经被禁止了，我们就需要使用其他的方式来实现串行访问。

在 RxJava 中，我们主要使用了两种串行访问实现方式，我称之为 **发射者循环（emitter-loop）**和**队列漏（queue-drain）**。（它们并不是我发明的，但我读过的大多数讲解 Java 并发编程的书中都没有提到这两种实现方式）

_注意：为了行文简洁，下文中我使用了 Java 8 的语法，但是 RxJava 需要保证在 Java 6 上运行，所以如果你要对 RxJava 的核心代码提交 pr，请确保你的语法以及**标准函数的调用（standard method calls）**只使用了 Java 6 的特性。_

## 发射者循环（emitter-loop）
之所以称这种实现方式为“发射者循环”，是因为我们使用了一个 `boolean` 值来记录当前是否有线程正在替其他所有线程（_译者注：只允许有一个线程执行发射操作，以此来保证串行访问_）进行发射操作，而正在发射的线程会把所有的发射任务执行完毕才会退出发射循环。

这种实现方式使用一个名为 `emitting` 的 `boolean` 值来记录是否有线程正在发射，而这个值的访问均在 `synchronized` 代码块中。

~~~ java
class EmitterLoopSerializer {
    boolean emitting;
    boolean missed;
    public void emit() {
        synchronized (this) {           // (1)
            if (emitting) {
                missed = true;          // (2)
                return;
            }
            emitting = true;            // (3)
        }
        for (;;) {
            // do all emission work     // (4) 
            synchronized (this) {       // (5)
                if (!missed) {          // (6)
                    emitting = false;
                    return;
                }
                missed = false;         // (7)
            }
        }
    }
}
~~~

下面我们分析一下 `emit()` 函数的工作机制：

1. 当我们进入 `synchronized` 代码块（1）时，可能存在两种状态：没有线程正在发射，或者有一个线程正在发射。由于我们在 `synchronized` 代码块中，如果（4）处有一个线程正在进行发射操作，那这个线程必须等待我们退出 `synchronized` 代码块（1），它才能进入 `synchronized` 代码块（5）。
2. 如果此时有一个线程正在发射，那我们就需要标记一下现在有更多的事件需要发射（_译者注：由于只能有一个线程执行发射操作，所以后来的线程就不能自己发射了，它需要告诉当前的发射者，还有更多事件需要发射_）。上面的例子中只是简单地使用了另一个 `boolean` 值来进行标记。出于不同的需求，我们可以使用不同的数据类型来标记还有更多的事件需要发射（例如 RxJava 的许多操作符使用了 `java.util.List`）。后面我们会对此有更多的介绍。
3. 如果此时没有线程正在发射，那当前线程就获得了执行发射操作的权利，它会把 `emitting` 置为 `true`。
4. 当一个线程获得执行发射操作的权利之后，我们就进入到了发射循环，并在此尽可能多的执行发射操作（_译者注：把这个线程能看到的所有需要发射的事件都发射出去_）。这个循环的具体实现取决于这个发射者循环需要完成的功能，但是必须非常小心地实现，否则就会导致**信号丢失和程序挂起（不执行）**。
5. 当发射循环 _认为_ 所有的发射任务执行完毕之后，它会进入 `synchronized` 代码块（5）。由于有可能会有其他线程在我们进入 `synchronized` 代码块之前调用 `emit` 函数，所以有可能依然还有事件需要发射。由于只有一个线程能进入 `synchronized` 代码块，加上我们使用的 `missed` 变量，所以当发射者循环进入 `synchronized` 代码块时，它只能看到仍然没有新的事件需要发射进而退出循环，或者又重新看到了新的事件进而继续循环发送。
6. 如果在发射者循环进入 `synchronized` 代码块（5）时，没有任何线程调用 `emit` 函数，我们就停止发射（把 `emitting` 置为 `false`）。新的线程在进入 `synchronized` 代码块（1）时，将会看到 `emitting` 为 `false` 了，所以这个线程就会自己进入发射者循环开始发射事件了。
7. 在发射者循环中，如果有更多的事件需要发射，我们会重置 `missed` 变量的值，然后重新开始循环。重置 `missed` 非常关键，否则将会导致死循环。

在实际的操作符中，我们有很多经典的方式来保存需要发射的事件（事件队列），供发射者循环发射。

方式之一就是使用具备线程安全性的数据结构来作为事件队列，因此对事件队列的访问就无需加锁了。

为了演示这种方式，下面我们看一下发射 `T` 类型数据的发射者循环版本：

~~~ java
class ValueEmitterLoop<T> {
    Queue<T> queue = new MpscLinkedQueue<>();    // (1)
    boolean emitting;
    Consumer<? super T> consumer;                // (2)

    public void emit(T value) {
        Objects.requireNonNull(value);
        queue.offer(value);                      // (3)
        synchronized (this) {
            if (emitting) {
                return;                          // (4)
            }
            emitting = true;
        }
        for (;;) {
            T v = queue.poll();                  // (5)
            if (v != null) {
                consumer.accept(v);              // (6)
            } else {
                synchronized (this) {
                    if (queue.isEmpty()) {       // (7)
                        emitting = false;
                        return;
                    }
                }
            }
        }
    }
}
~~~

在 `ValueEmitterLoop` 中，实现逻辑和第一个例子略有不同：

1. 我们使用了一个线程安全的队列来保存需要发射的事件。由于可能会有很多个线程调用 `emit` 函数（向队列中添加数据），而只会有一个线程从队列中取出数据，所以 Java 的 `ConcurrentLinkedQueue` 会有不必要的性能开销。我建议使用 [JCTools](https://github.com/JCTools/JCTools){:target="_blank"} 的优化版本。例子中使用了无限长度的变体版本，但如果可能的最大队列长度是已知的（有可能发射者循环需要实现的功能中，backpressure 策略会为队列长度设置一个上限），那就可以使用 `MpscArrayQueue` 了。
2. 例子中我们使用 `Consumer` 回调来发射事件。
3. 首先我们向队列中添加非 `null` 的数据（JCTools 不支持 `null` 值）。
4. 接下来我们进入到 `synchronized` 代码块中，如果当前有其他线程正在发射，那我们就结束执行。这里我们就无需 `missed` 变量了，因为队列是否为空可以用来判断是否还有更多事件需要发射。
5. 在循环中，我们从队列中取出数据。
6. 由于我们的队列中不允许有 `null`，所以从队列中取出 `null` 就意味着队列已经空了。
7. 进入第二个 `synchronized` 代码块之后，我们再次检查队列是否为空，如果为空，我们就把 `emitting` 置为 `false` 并退出执行。由于我们向队列中添加元素的代码在 `synchronized` 代码块之外，所以在发射者线程从队列中取到 `null`（5）之后而执行到（7）之前，可能会有线程向队列中添加新的数据。可能会有两种情况：a) 在执行到（7）之前有新的数据被加入到队列中，这种情况下发射者线程就会继续执行发射循环；b) 新的数据在发射者结束执行（7）之后才被加入到队列中，这种情况下，就会有新的线程进入发射者循环，进而开始持续发射事件。总的来说，不会发生信号丢失。

第二种保存待发射事件队列的方式是在 `synchronized` 代码块中对队列进行操作，所以我们可以使用非线程安全的数据结构。

而这种方式正是 RxJava 中 `SerializedObserver` 的实现方式。接下来我对上面的例子稍作修改：

~~~ java
class ValueListEmitterLoop<T> {
    List<T> queue;                           // (1)
    boolean emitting;
    Consumer<? super T> consumer;

    public void emit(T value) {
        synchronized (this) {
            if (emitting) {
                List<T> q = queue;
                if (q == null) {
                    q = new ArrayList<>();   // (2)
                    queue = q;
                }
                q.add(value);
                return;
            }
            emitting = true;
        }
        consumer.accept(value);              // (3)
        for (;;) {
             List<T> q;
             synchronized (this) {           // (4)
                 q = queue;
                 if (q == null) {            // (5)
                     emitting = false;
                     return;
                 }
                 queue = null;               // (6)
             }
             q.forEach(consumer);            // (7)
        }        
    }
}
~~~ 

`ValueListEmitterLoop` 在 `synchronized` 代码块中的逻辑稍显复杂，但它的行为还是很直观的：

1. 我们使用了 `java.util.List` 来保存待发射的事件，此外我们还用它是否为 `null` 来标记是否有待发射事件。一开始是没有待发射事件的，所以 `queue` 为 `null`。
2. 如果我们发现此时有线程正在发射，我们就把数据添加到队列中（如果此前没有待发射事件，那我们就新建一个 `ArrayList`）。
3. 获得发射权利的线程现在就可以直接向 `consumer` 发射当前的事件（3）。当前的事件无需添加到 `queue` 中（可以节省时间），而且此时也无需检查 `queue` 中的元素，因为此时它必定为 `null`（由于上一个发射者循环退出的条件就是 `queue` 为 `null`）。
4. 当前事件发射完毕之后，此时就可能有待发射事件了，所以我们进入 `synchronized` 代码块检查 `queue` 是否不为 `null`。
5. 如果 `queue` 为 `null`，那我们就退出。由于 `queue` 的创建或者添加元素是在另一个 `synchronized` 代码块中进行的，所以不存在竞争问题，也不会发生信号丢失。
6. 我们把 `queue` 置为 `null`，表明（当前）没有更多的事件需要发射了（_译者注：我们把所有待发射的事件都取出到 `q` 中了_）。这就保证当我们再次开始循环之后，如果没有线程调用 `emit()` 函数，循环将在（5）检查之后结束。
7. 我们直接对 `q` 执行了 `forEach` 操作，这样做是线程安全的，因为我们把 `q` 从类中“摘除”了，没有其他线程可以访问它。

不幸的是，调用 `consumer.accept()` 的时候，可能会有 unchecked exception 被抛出，然后我们的发射者循环就退出了，但是 `emitting` 值依然为 `true`。如果发射事件的过程中伴随着 error 事件（参考 `Notification` 和 `NotificationLite`），那这种情况就会很常见。这种情况下，一旦异常被抛出之后，紧接着就会有新的线程调用 `emit()` 函数（_译者注：但此时由于 `emitting` 值依然为 `true`，新的线程并不会进入发射者循环，就出问题了_）。

为了避免这种情况，我们可以为每个调用都加上 `try-catch`，但通常 `emit()` 的调用方还是需要知道有异常发生的。所以我们可以把异常传播出去（propagate out），然后在异常发生时把 `emitting` 置为 `false`。

~~~ java
    // same as above
    // ...
    public void emit(T value) {
        synchronized (this) {
            // same as above
            // ...
        }
        boolean skipFinal = false;             // (1)
        try {
            consumer.accept(value);            // (5)
            for (;;) {
                List<T> q;
                synchronized (this) {           
                    q = queue;
                    if (q == null) {            
                        emitting = false;
                        skipFinal = true;      // (2)
                        return;
                    }
                    queue = null;
                }
                q.forEach(consumer);           // (6)
            }
        } finally {
            if (!skipFinal) {                  // (3)
                synchronized (this) {
                    emitting = false;          // (4)
                }
            }
        }
    }
~~~

由于在 Java 6 中我们不能在 `catch` 语句中把 `Throwable` 再次抛出，所以我们需要在 `finally` 语句中执行我们的重置 `emitting` 逻辑。

1. 我们定义了一个 `skipFinal` 变量，如果为 `true`，就表明我们正常退出循环，并跳过 `finally` 语句中的逻辑。
2. 如果 `queue` 为 `null`，我们设置 `skipFinal` 为 `true`，所以（3）处的检查就不会通过，因此就跳过了 `finally` 语句中的 `synchronized` 代码块。之所以需要这样做，是因为如果我们无条件地把 `emitting` 置为 `false`，那就有可能在当前线程执行于（2）和（3）之间的时候，让其他线程通过 `emit()` 函数开头的 `synchronized` 代码块（_译者注：这就无法保证只有一个线程处于发射者循环中了_）。
3. （5）或者（6）的 `accept` 调用如果抛出了异常，`skipFinal` 就会是 `false`，所以就会执行 `finally` 语句中的代码，把 `emitting` 置为 `false`。`finally` 语句执行完毕之后，异常会继续抛出到 `emit()` 函数的调用者那里。
4. （发生异常之后）我们把 `emitting` 置为了 `false`，这就让后续的 `emit()` 函数调用可以正确执行了。

你可能听说过在并发代码中使用 `synchronized` 会降低性能，但为什么我们仍然在 RxJava 中如此频繁地使用它呢？

总的来说，**第一原则是在假定存在性能问题之前，进行性能测量**。我们对 RxJava 进行了测量，结果出乎意料：`synchronized` 反而带来了更大的吞吐量。我也许会把 benchmarks 发布出来。

由于 RxJava 无法断定使用者的多线程或者线程调度场景，所以它需要在同步和异步场景下都能很好的工作。而我们发现，大部分应用在使用 RxJava 时，都是在同步场景中。

现在的 Java（虚拟机） 对同步和单线程操作的代码进行了特殊的性能优化，例如**偏向锁（biased locking）**和**锁省略（lock elision）**。在上面的例子中，由于这两种技术，`synchronized` 操作会被移除，这样就完全没有额外的性能开销了，所以在串行场景下性能会更优（有的场景下优势是非常巨大的）。

相反，其他方式中对无锁化原子类的使用（例如我将在下一篇讲解的队列漏），会带来无法避免的额外开销，而且如果涉及到复杂的情况，还会产生更多的内存分配。

当然，如果 Java（虚拟机）检测到了这些锁上的并发操作，那偏向锁就会被移除，代码又会重新按照常规的加锁方式执行（值得注意的是，移除偏向锁的操作将会 stop-the-world，所以会带来巨大的延迟）。

这种潜在的性能提升是很难取得的，因此 RxJava 最终还是选择了 `synchronized`。

## 总结
在编写 RxJava 操作符时需要理解的最重要的算法，就是如何把并发的事件发射转化为串行的事件发射。在本文中，我介绍了一种叫做**发射者循环（emitter-loop）**的方式，这种方式和其他方式相比，具有不错的性能表现。

由于这种方式使用了锁进行同步，以及偏向锁移除时的巨大开销，我建议只在异步运行比例低于 50% 的场景下使用这种方法。

然而，如果异步运行比例高于 50%，或者可以确定一定会涉及到多线程（例如 `observeOn`），那就还有一种性能可能更好的方式，我称之为队列漏，我将在下一篇中进行讲解。
