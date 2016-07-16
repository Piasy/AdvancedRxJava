---
layout: post
title: Operator 并发原语： subscription-containers（一），标准容器类与 TwoSubscribers
tags:
    - Operator
    - Subscription
---

原文 [Operator concurrency primitives: subscription-containers (part 1)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_19.html){:target="_blank"}

## 介绍

编写复杂的操作符通常涉及到 subscription 管理以及一系列的 subscription 容器。

有一点值得重申：subscription 容器都有一个终结状态，`unsubscribe()` 函数被调用之后它们就处于终结状态了，此后增加或者替换 subscription 时，都会取消订阅这个新的 subscription。这一点看起来可能有些奇怪，但接下来描述的一些场景将会解释这一设计决策背后的原因。

假设你的操作符用 `Scheduler.Worker` 实例调度了一个延迟任务，此时并发的线程取消订阅了整个（_Observable_）链条，而 `schedule()` 函数还没有返回 `Subscription`，所以你也就无从取消这个延迟任务。如果订阅容器没有终结状态属性，那么把（_`schedule()` 函数返回的_）`Subscription` 加入到容器之后，这个容器就永远也不会取消订阅这个 subscription 了（_因为使用方已经调用过 `unsubscribe()` 了，他不会再调用了_），这时就发生了内存泄漏。有了终结状态属性之后，如果我们尝试把（_`schedule()` 函数返回的_）`Subscription` 加入到容器中，容器会立即取消订阅这个 subscription，相关资源也就立即释放了。这一点在安卓开发领域广为人知，因为在 app 暂停之后，我们会希望取消（_Observable_）链条中的后台任务。

还有其他一些我们早已知晓的特性：

+ 所有对容器的操作，从并发取消订阅的角度来说，都必须是线程安全和原子性的；
+ `unsubscribe()` 必须是幂等的（多次调用没有任何作用）；
+ 我们应该避免在持有锁时调用 `unsubscribe()`（尤其是调用方有权持有锁时），以避免任何潜在的死锁情况。

在本文中，我将简要介绍这些容器类，它们在操作符中的使用，以及值得注意的属性。但本文更深刻的价值是介绍这些容器类背后的概念和思想，以帮助大家开发自定义的容器类，以支持在 `Subject` 以及诸如 `publish()` 的转换操作符中的复杂生命周期。

## 标准 subscription 容器类

### CompositeSubscription

用得最多的容器就是 `CompositeSubscription` 了。任何需要和资源（`Subscriber` 或者 `Future` 包装类型）打交道的操作符，通常都会选择这个容器。当前的实现版本使用了 `synchronized` 代码块来保证线程安全，并把 subscription 保存在 `HashSet` 中。

（_注：之所以不用 copy-on-write 的非阻塞方式，是由于 copy-on-write 会产生很多垃圾对象（很快被垃圾回收的对象），而在 Netflix 对 RxJava 的一些特定使用场景让他们比较担心 GC 的性能。_）

一个值得注意的特性是，当我们调用 `remove()` 从中移除一个 subscription 时，如果这个 subscription 确实在容器中，那这个 subscription 会被取消订阅。这一点有可能并非我们想要的效果，例如我们有一个 `Future` 的包装类型放入了容器中，我们希望在它执行完毕之后从容器中移除，此时我们可能并不希望它被取消订阅。

有些操作符它们需要处理的资源（_的添加和移除_）具有不确定性，对这些操作符来说 `CompositeSubscription` 就很合适，例如 `merge()`，此外其内部的 `HashSet` O(1) 的操作复杂度在性能上也很有优势。

### SerialSubscription

`SerialSubscription` 同时只能包含一个 subscription，在设置新的 subscription 时它会取消订阅老的 subscription。在其内部它使用了 copy-on-write 方式加上一个状态变量来实现。

这种容器主要用于需要每次关注一个资源的情形，例如 `concatMap()` 中当前的 `Subscriber`。

### MultipleAssignmentSubscription

`MultipleAssignmentSubscription` 是 `SerialSubscription` 的一个变体（尽管类继承关系不一样），它也用于一次只包含一个 subscription，但是设置新的 subscription 时不会取消订阅老的 subscription。

由于这一特性，它的使用场景更少，也需要更加谨慎。只有当它包含的那个 subscription 可以轻易“丢失”时，也就是说调用一次 `unsubscribe()` 是多余的，我们才可以使用它。这一容器被应用于通过 `Scheduler.Worker` 来实现定期调度的算法，在算法中我们会发出独立的延迟调度。`schedulePeriodically()` 返回的类型就是 `MultipleAssignmentSubscription`，它会包含当前的延迟调度。由于我们是在每个延迟调度结束的时候替换新的 subscription，所以我们没必要甚至不想要取消订阅老的 subscription。

### SubscriptionList

`SubscriptionList` 是一个内部的实现，它和 `CompositeSubscription` 类似，只不过使用 `LinkedList` 保存 subscription 而不是 `HashSet`，这样可以让只有添加没有移除的场景更加高效。通常来说开发者不应该依赖内部实现，但如果你想要给 RxJava 提交 PR，我希望在合适的情况下你能使用这个容器类（而且它此时依然可以使用）。

最近，这个容器类加上了 `remove()` 方法，以支持对默认 `computation()` 调度器的优化，加速对无延迟任务的添加以及移除，因为通常来说这些任务会按照同样的顺序执行与移除。所以我们每次移除的都是第一个元素，这比从 `HashMap` 移除元素效率要高（根据我们的 benchmark）。

这一容器类出现在多处，最多的就是在 `Subscriber` 类中，但内部的 `ScheduledAction` 同样也使用了它。

### RefCountSubscription

在我看来 `RefCountSubscription` 是 Rx.NET 中一系列资源容器的一个遗民。它只会持有一个不能修改的 `Subscription`，并且“派发”从中派生出来的 subscription，而且当所有的派生 subscription 被取消订阅之后，它持有的 subscription 才会被取消订阅。我们在 RxJava 中很长时间都没有使用它，而在 Rx.NET 中，`RefCountDisposable` 工作得要比其他资源管理的 `IDisposables` 要好得多。

（_注：`BooleanSubscription` 不是一个容器类型，因为它没有持有其他 `Subscription` 的引用，它是用来容纳那些将要在取消订阅时被执行的任务的。_）

## 实现阻塞的容器类型

假设我们需要这样一个容器类型，它只能容纳两个 subscription，而且可以随时被替换（替换时老的 subscription 会被取消订阅）。让我们称之为 `TwoSubscribers`，它的结构如下：

~~~ java
public final class TwoSubscribers 
implements Subscription {
    private volatile boolean isUnsubscribed;          // (1)
     
    Subscription s1;                                  // (2)
    Subscription s2;
     
    @Override
    public boolean isUnsubscribed() {
        return isUnsubscribed;                        // (3)
    }
     
    public void set(boolean first, Subscription s) {
        // implement
    }
     
    @Override
    public void unsubscribe() {
        // implement
    }
}
~~~

现在它看起来并不复杂。我们用 `volatile boolean` 来记录当前是否已经被取消订阅，以避免（1）（3）处不必要的 `synchronized` 代码块。这个类需要容纳两个 subscription，由于我们依赖外部的同步控制，所以我们直接使用了简单的成员变量（2）。

`set()` 函数接收一个 `boolean` 值来表明是哪个 subscription 需要被替换。它的实现如下：

~~~ java
// ...
public void set(boolean first, Subscription s) {
    if (!isUnsubscribed) {                       // (1)
        synchronized (this) {
            if (!isUnsubscribed) {               // (2)
                Subscription temp;               // (3)
                if (first) {
                    temp = s1;
                    s1 = s;
                } else {
                    temp = s2;
                    s2 = s;
                }
                s = temp;                        // (4)
            }
        }
    }
    if (s != null) {                             // (5)
        s.unsubscribe();
    }
}
// ...
~~~

不像一直以来复杂的 `Producer`，这里代码看起来很简单：

1. 我们提前检查是否已经被取消订阅，如果是那就可以跳过 `synchronized` 代码块，直接取消订阅参数即可。
2. 否则我们二次确认是否依然没有被取消订阅，如果没有被取消订阅，那我们就开始替换的逻辑。
3. 由于老的 subscription 需要被取消订阅，所以我们把它和参数进行交换。
4. 我们复用传入参数，让我们后面直接取消订阅 `s` 即可。
5. 我们可能从（1）（2）处执行到此，也可能在进行了替换之后执行到此，由于后者可能为 `null`，所以如果 `s` 不为 `null`，我们就取消订阅 `s`。

现在就剩下 `unsubscribe()` 了：

~~~ java
// ...
@Override
public void unsubscribe() {
    if (!isUnsubscribed) {                  // (1)
        Subscription one;                   // (2)
        Subscription two;
        synchronized (this) {
            if (isUnsubscribed) {           // (3)
                return;
            }
                
            isUnsubscribed = true;          // (4)
                
            one = s1;                       // (5)
            two = s2;
                
            s1 = null;
            s2 = null;
        }
            
        List<Throwable> errors = null;      // (6)
        try {
            if (one != null) {
                one.unsubscribe();          // (7)
            }
        } catch (Throwable t) {
            if (errors == null) {
                errors = new ArrayList<>(); // (8)
            }
            errors.add(t);
        }
        try {
            if (two != null) {
                two.unsubscribe();
            }
        } catch (Throwable t) {
            if (errors == null) {
                errors = new ArrayList<>();
            }
            errors.add(t);
        }

        Exceptions.throwIfAny(errors);      // (9)
    }
}
~~~

不幸的是，这个方法看起来没那么优雅，不过我们不得不这样：

1. 我们检查容器是否已经被取消订阅了，如果是那我们就不用做任何事了。
2. 如果容器还没有被取消订阅，那我们就需要把当前引用的 subscription 摘出来，因为我们**不能在持有锁的时候取消订阅**（_译者注：还记得文章开头说的几点注意事项吗？_）。
3. 在 `synchronized` 代码块中，我们二次确认是否依然没有被取消订阅。
4. 我们尽早设置 `isUnsubscribed` 为 `true`，因为这样一来并发的 `set()` 调用就能尽早看到 `isUnsubscribed` 的新值，从而就不必等待进入 `synchronized` 代码块了。
5. 我们取出老的 subscription，并把成员置为 `null`，避免滞留它们的引用。
6. 谨防 `unsubscribe()` 函数中可能抛出的异常是一个很好的实践，而由于我们有两个 subscription，所以我们需要（_两个 try-catch 来_）保证它们都能被取消订阅并且异常也能被抛出。
7. 如果老的 subscription 不为 `null`，我们就将其取消订阅。
8. 如果发生异常，我们就创建一个异常列表，并把异常加入进去。
9. 在对两个 subscription 都执行了相同的逻辑之后，我们就利用一个辅助方法来抛出存在的异常，如果两个 subscription 的取消订阅过程都抛出了异常，那 `throwIfAny` 将会抛出一个包含两个异常的 `CompositeException`。

## 总结

本文相对来说比较简短，我在本文中回顾了已经存在的几种标准容器类型，包括它们内在的要求，属性，以及它们的使用要点。同时我也演示了如何通过阻塞同步操作实现一个和标准容器类型具有一致性的容器类型。

在下一篇中，我将演示如何实现一个非阻塞版本的 `TwoSubscriptions`，并且将其扩展为可以容纳任意数量的 subscription 且保持非阻塞特性。
