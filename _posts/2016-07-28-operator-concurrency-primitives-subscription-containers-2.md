---
layout: post
title: Operator 并发原语： subscription-containers（二），无锁化 TwoSubscribers
tags:
    - Operator
    - Subscription
---

原文 [Operator concurrency primitives: subscription-containers (part 2)](http://akarnokd.blogspot.com/2015/05/operator-concurrency-primitives_22.html){:target="_blank"}

## 介绍

在本文中，我将实现[前文中](/AdvancedRxJava/2016/07/15/operator-concurrency-primitives-subscription-containers-1/){:target="_blank"}介绍的 `TwoSubscribers` 的两种无锁化（_非阻塞_）版本。尽管它们在功能上完全一致，但是在实现的过程中将表现出在处理订阅状态检查和取消订阅时两种不同的哲学。

## 在状态类中使用 `boolean isUnsubscribed`

要实现一个无锁化的线程安全数据结构，一种简单地办法就是在每次修改时进行所谓的 “copy-on-write” 操作，即利用一个不可变的状态变量加上一个 CAS 循环。在这里，我们利用一个组合类来包含两个不同的 `Subscription`：

~~~ java
static final class State {
    final Subscription s1;
    final Subscription s2;
    final boolean isUnsubscribed;
    public State(Subscription s1, 
            Subscription s2, 
            boolean isUnsubscribed) {
        this.s1 = s1;
        this.s2 = s2;
        this.isUnsubscribed = isUnsubscribed;
         
    }
}
// ...
~~~

有了 `State` 类之后，我们需要改变状态时，会先创建一个新的 `State` 对象（_为其设置相应的值_），再利用一个 CAS 循环保证原子性。现在让我们来看看新的无锁化容器类结构：

~~~ java
public final class TwoSubscribersLockFree1 
implements Subscription {
    static final class State { 
        // ...
    }
 
    static final State EMPTY = 
        new State(null, null, false);                  // (1)
 
    static final State UNSUBSCRIBED = 
        new State(null, null, true);                   // (2)
     
    final AtomicReference<State> state = 
        new AtomicReference<>(EMPTY);                  // (3)
 
    public void set(boolean first, Subscription s) {
        // implement
    }
     
    @Override
    public void unsubscribe() {
        // implement
    }
 
    @Override
    public boolean isUnsubscribed() {
        // implement
    }
}
~~~

首先，任何情况下初始状态和已经取消订阅的状态都是不变的，所以我们把它们声明为 `static final` 常量，它们只有 `isUnsubscribed` 值不同（1）（2）。由于状态的切换需要具备原子性，所以我们使用 `AtomicReference` 来引用 `State` 成员（3），它被初始化为初始状态。

在上述结构之上，`set()` 的实现如下：

~~~ java
public void set(boolean first, Subscription s) {
    for (;;) {
        State current = state.get();                    // (1)
        if (current.isUnsubscribed) {                   // (2)
            s.unsubscribe();
            return;
        }
        State next;
        Subscription old;
        if (first) {
            next = new State(s, current.s2, false);     // (3)
            old = current.s1;                           // (4)
        } else {
            next = new State(current.s1, s, false);
            old = current.s2;
        }
        if (state.compareAndSet(current, next)) {       // (5)
            if (old != null) {
                old.unsubscribe();                      // (6)
            }
            return;
        }
    }
}
~~~

1. 读取当前状态。
2. 如果当前状态已经被取消订阅，说明我们已经到了终结状态，所以我们直接取消订阅 `s` 之后返回。
3. 否则，我们就基于当前状态创建一个新的状态，替换相应的 subscription。
4. 由于被替换的 subscription 需要被取消订阅，所以我们用一个局部变量保存被替换的 subscription。
5. 我们通过 CAS 操作来切换新旧状态，如果失败，说明当前有并发线程成功修改了状态，所以我们继续循环进行尝试。
6. 如果 CAS 成功且被替换的 subscription 不为 `null`，我们就将其取消订阅。

`isUnsubscribed()` 的实现非常直观：

~~~ java
// ...
@Override
public boolean isUnsubscribed() {
    return state.get().isUnsubscribed;
}
// ...
~~~

最后我们看一下 `unsubscribe()` 的实现：

~~~ java
    @Override
    public void unsubscribe() {
        State current = state.get();                        // (1)
        if (!current.isUnsubscribed) {                      // (2)
            current = state.getAndSet(UNSUBSCRIBED);        // (3)
            if (!current.isUnsubscribed) {                  // (4)
                List<Throwable> errors = null;              // (5)
                 
                errors = unsubscribe(current.s1, errors);   // (6)
                errors = unsubscribe(current.s2, errors);
                 
                Exceptions.throwIfAny(errors);              // (7)
            }
        }
    }
 
    private List<Throwable> unsubscribe(Subscription s,     // (8)
            List<Throwable> errors) {
        if (s != null) {
            try {
                s.unsubscribe();
            } catch (Throwable e) {
                if (errors == null) {
                    errors = new ArrayList<>();
                }
                errors.add(e);
            }
        }
        return errors;
    }
}
~~~

其中有几点值得一提：

1. 我们读取当前的状态。
2. 如果当前状态就已经被取消订阅，那我们就什么也不用做了。
3. 否则我们用原子操作把状态设置为终结状态。
4. 如果更新之前的状态已经被取消订阅，那我们就可以返回了，否则，由于 `getAndSet` 是原子操作，只会有一个调用者成功进行了状态切换。这里无需进行 CAS 循环，得益于平台支持的 `getAndSet` 操作，我们已经实现了非阻塞。
5. 可能的异常被收集到 `errors` 中。
6. 我重构了取消订阅的代码，把取消订阅一个 subscription 的逻辑封装到函数中（_译者注：把函数调用的返回值赋值给 `errors` 非常重要！_）。
7. 如果取消订阅时发生了异常，我们就将其抛出。
8. 我们在一个辅助函数中取消订阅，并捕获异常。

## 使用 `UNSUBSCRIBED` 状态的引用

试想，终结状态除了 `isUnsubscribed` 变量的值之外，和其他状态依然是可以区分的（通过一个常量引用）。所以我们可以把 `State` 类中的 `isUnsubscribed` 变量去掉，在需要的地方直接对比当前状态和 `UNSUBSCRIBED` 状态的引用。

所以我们的 `TwoSubscribersLockFree2` 中 `State` 变成了：

~~~ java
public final class TwoSubscribersLockFree2 implements Subscription {
    static final class State {
        final Subscription s1;
        final Subscription s2;
        public State(Subscription s1, 
                Subscription s2) {
            this.s1 = s1;
            this.s2 = s2;
            
        }
    }
~~~

由于我们去掉了 `isUnsubscribed` 变量，所以我们需要修改之前对它进行检查的代码：

~~~ java
// ...
static final State EMPTY = new State(null, null);         // (1) 
static final State UNSUBSCRIBED = new State(null, null);

final AtomicReference<tate> state
    = new AtomicReference<>(EMPTY);

public void set(boolean first, Subscription s) {
    for (;;) {
        State current = state.get();
        if (current == UNSUBSCRIBED) {                    // (2)
            s.unsubscribe();
            return;
        }
        State next;
        Subscription old;
        if (first) {
            next = new State(s, current.s2);
            old = current.s1;
        } else {
            next = new State(current.s1, s);
            old = current.s2;
        }
        if (state.compareAndSet(current, next)) {
            if (old != null) {
                old.unsubscribe();
            }
            return;
        }
    }
}

@Override
public boolean isUnsubscribed() {
    return state.get() == UNSUBSCRIBED;                    // (3)
}

@Override
public void unsubscribe() {
    State current = state.get();
    if (current != UNSUBSCRIBED) {                         // (4)
        current = state.getAndSet(UNSUBSCRIBED);
        if (current != UNSUBSCRIBED) {                     // (5)
            List<Throwable> errors = null;
            
            errors = unsubscribe(current.s1, errors);
            errors = unsubscribe(current.s2, errors);
            
            Exceptions.throwIfAny(errors);
        }
    }
}
// ...
~~~

初始状态和终结状态再也不需要一个 `boolean` 变量了（1），而所有的 `current.isUnsubscribed` 都被替换为了 `current == UNSUBSCRIBED`（2，3，4，5）。

这两种方式我们应该选择哪一种呢？运行一下 Benchmark 然后自己决定吧！显然，第一种方式分配了更多的内存，但在有些平台上，检查 `boolean` 更快，而第二种方式占用更少的内存，但是引用的对比相对慢一些。

通常来说，使用上面的实现会增加 GC 的压力，因为每次修改状态都会进行新状态的创建。我们可以针对每个 subscription 进行单独的 CAS 循环，这样就不需要额外的 `State` 类了，也就不需要额外的对象创建了，但一旦 subscription 数量变多，代码就会变得非常冗长。

## 总结

在本文中，我介绍了 `TwoSubscribers` 的两种无锁化版本，并对其工作原理进行了讲解。

但通常我们很可能需要管理不止两个 subscription，所以下篇文章中我将展示一种基于数组的容器类，并解析其工作原理。
