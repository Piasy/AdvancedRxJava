---
layout: post
title: Operator 并发原语： subscription-containers（三，完结），基于数组的容器类
tags:
    - Operator
    - Subscription
---

原文 [Operator concurrency primitives: subscription-containers (part 3 - final)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_26.html){:target="_blank"}

## 介绍

在最后一篇关于 subscription 容器类的文章中（_本文_），我将介绍一种基于数组的 copy-on-write 线程安全容器类。

为什么这一容器类型如此重要？让我反问一句：如果被包含的 subscription 是 `Subscriber`，我们如何处理 `Subscriber` 数组？

你可以实现一个支持给多个子 Subscriber 组播的操作符，同时保证线程安全性和终结状态，就像 `Subject` 处理其 Subscriber 一样，以及像最近（_2015年5月_）对 `publish()` 的重新实现那样。

## 基于数组的容器类

让我们实现这样一个容器类：

+ 能添加和移除某种 subscription，例如 `Subscriber`；
+ 能获取当前的内容（_包含的 subscription_）；
+ 能够在不取消订阅其包含的 subscription 的前提下，取消订阅容器类；
+ 添加 subscription 能得知是否成功；

它的类结构如下：

~~~ java
@SuppressWarnings({"rawtypes", "unchecked"})                 // (1)
public class SubscriberContainer<T> {
    static final Subscriber[] EMPTY = new Subscriber[0];     // (2)
    static final Subscriber[] TERMINATE = new Subscriber[0];
     
    final AtomicReference<Subscriber[]> array
        = new AtomicReference<>(EMPTY);                      // (3)
     
    public Subscriber<T>[] get() {                           // (4)
        return array.get();
    }
 
    public boolean add(Subscriber<T> s) {                    // (5)
        // implement
    }
 
    public boolean remove(Subscriber<T> s) {                 // (6)
        // implement
    }
 
    public Subscriber<T>[] getAndTerminate() {               // (7)
        return array.getAndSet(TERMINATE);
    }
 
    public boolean isTerminated() {                          // (8)
        return get() == TERMINATED;
    }
}
~~~

它包含以下几个元素：

1. 由于 Java 的数组不支持泛型，并且特定的类型转换不符合规范，所以我们必须掩盖 rawtypes 以及 unchecked 类型转换（为泛型类型）的警告。通常来说，我们的实现是安全的（_不存在类型转换失败_），但我们需要让用户的代码具备类型安全性。
2. 我们用数组常量来表示初始状态和终结状态。
3. 我们用 `AtomicReference` 来引用数组，如果我们知道这个容器类不会被继承，那我们可以直接继承自 `AtomicReference`。
4. `get()` 函数返回当前包含的元素。处于性能考虑，该函数的返回值只能用于读访问（否则就需要在每次返回之前进行深拷贝）。
5. `add()` 函数接收带有类型的 `Subscriber`，如果添加成功就返回 `true`，否则如果容器类已经被取消订阅，就返回 `false`。
6. `remove()` 尝试移除指定的 `Subscriber`，如果移除成功就返回 `true`。
7. 这里没有 `unsubscribe()` 函数，而是取了个巧：我们把容器类的数组替换为终结状态，并把原来的值返回。当我们需要原子性终结，但在终结之后需要进行其他操作时，这一方式就很有用了。
8. 由于一个空的数组不能表明处于终结状态（无法和初始状态进行区分），所以我们需要和 `TERMINATED` 对比引用。

`add()` 很简单：

~~~ java
// ...
public boolean add(Subscriber<T> s) {
    for (;;) {
        Subscriber[] current = array.get();
        if (current == TERMINATED) {                  // (1)
            return false;
        }
        int n = current.length;
        Subscriber[] next = new Subscriber[n + 1];
        System.arraycopy(current, 0, next, 0, n);     // (2)
        next[n] = s;
        if (array.compareAndSet(current, next)) {     // (3)
            return true;
        }
    }
}
// ...
~~~

这里又是一个典型的 CAS 循环：

1. 如果容器类已经处于终结状态，那我们就直接返回 `false`，待添加的 Subscriber 也不会被添加。调用方可以根据返回值决定如何处理待添加的 Subscriber，例如向其发送 `onCompleted` 事件。
2. 我们为数组扩容，拷贝已经存在的 Subscriber，并把新的添加在最后。
3. CAS 操作相当于把修改进行一次提交操作，如果成功，我们就返回 `true`，否则我们就继续尝试。

最后是 `remove()` 函数：

~~~ java
// ...
public boolean remove(Subscriber<T> s) {
    for (;;) {
        Subscriber[] current = array.get();
        if (current == EMPTY 
                || current == TERMINATED) {             // (1)
            return false;
        }
        int n = current.length;
        int j = -1;
        for (int i = 0; i < n; i++) {                   // (2)
            Subscriber e = current[i];
            if (e.equals(s)) {
                j = i;
                break;
            }
            i++;
        }
        if (j < 0) {                                    // (3)
            return false;
        }
        Subscriber[] next;
        if (n == 1) {                                   // (4)
            next = EMPTY;
        } else {
            next = new Subscriber[n - 1];
            System.arraycopy(current, 0, next, 0, j);
            System.arraycopy(current, j + 1, 
                next, j, n - j - 1);                    // (5)
        }
        if (array.compareAndSet(current, next)) {       // (6)
            return true;
        }
    }
}
~~~

尽管代码看起来有点复杂，但是它的逻辑还是很直观的：

1. 如果当前数组是空的，或者容器类已经终止了，那我们就可以直接返回 `false` 了。
2. 否则我们就从前往后搜索到第一个要移除的 Subscriber（用 `equals` 判断），并把它的下标记为 `j`。（_这里省略了一句原文，“By scanning first instead of doing an on-the-fly filter-copy, we can save some overhead due to the card-marking associated with each reference store, required by most GCs”，和作者沟通之后，大意就是：“我们先查找，再分段复制，而不是先创建一个数组，再逐个判断复制，这样可以快一些，性能差异还涉及到一些关于引用存储相关的原理”，先查找再分段复制，大家应该都会这么干，不然你新建一个多大的数组呢？都不知道能不能找到呢！但是引用存储的原理什么的，这里我就不懂，暂时也不想懂了，有兴趣可以自行扩展_）。
3. 查找结束之后，如果 `j` 是负数，说明指定的 Subscriber 不在数组中，所以我们返回 `false`。
4. 如果原数组就只有一个元素，那我们就无需创建一个新的空数组，我们可以直接复用 `EMPTY` 常量（因为空数组没有状态，意义都是一样的）。
5. 如果原数组不止一个元素，那我们就创建一个长度减一的新数组，并把目标周围的元素都复制过去。
6. 最终 CAS 操作会尝试替换新的数组，如果成功就返回 `true`，这意味着我们成功移除了指定的 Subscriber。

像上面这种容器，在类似于组播的操作符中用得比较多，但是这些操作符的使用场景中，基本不会涉及到超过 6 个 Subscriber，所以频繁的创建数组对象影响并不大。

如果在你的使用场景下数组创建比例很高进而产生了性能问题，那你可以把上述逻辑改成基于 `synchronized` 块的实现，并使用 `List` 或者 `Set` 这样的数据结构存储 Subscriber，但要注意在派发事件时会被频繁调用的 `get()` 方法，这时就不能用非阻塞的方式实现了。这时 `get()` 很可能需要利用 `synchronized` 块实现，并且还需要进行 defensive copy，所以你需要**仔细权衡考虑**。

## 总结

在这个迷你系列中，我介绍了几种标准的 subscription 容器类型，并且展示了如何实现阻塞和非阻塞的自定义容器。最后我展示了一种基于数组的非阻塞容器，它的一些看似奇特的特性在组播类操作符中非常有用。

如果我们只有 RxJava 1.x，那操作符并发原语（operator concurrency primitives）系列就可以结束了。但 reactive-streams 标准最近已经定稿，而它将是 RxJava 2.x 的基石，所以这些并发原语都需要完全重写，无法避免。

这是否意味着前面这么多的东西都是白学的？是也不是。实际上涉及到的一些理念都是相通的，只是类的结构需要根据 reactive-streams 规范进行调整。

在开始 RxJava 2.x 相关的内容之前，让我们稍事休息，先看看其他进阶话题的内容：**调度器（schedulers）**。
