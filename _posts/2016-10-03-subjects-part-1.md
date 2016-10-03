---
layout: post
title: Subjects（一）：Subject 概念和 RxJava 中的标准实现
tags:
    - Subject
---

原文 [Subjects (part 1)](http://akarnokd.blogspot.com/2015/06/subjects-part-1.html){:target="_blank"}

## 介绍

我猜有很多人都恨死 `Subject` 了，但我还是要写一个关于它的系列文章。

有些人觉得它是响应式编程世界里面的可变状态，但我并不这样认为，然后他们就进一步叫嚣：_不要使用 `Subject`，而是更多地使用 `Observable.create()`：_

~~~ java
Observable.create(s -> {
   int i = 0;
   while (true) {
       s.onNext(i++);
   }
}).subscribe(System.out::println);
~~~

那我就有问题了，取消订阅和 backpressure 怎么处理？在上面的代码中，没有操作符能解决这些问题，但我们可以在任何时间对 `Subject` 使用 `onBackpressureXXX` 操作符。

`Subject` 是无法被使用者搞破坏的，但响应式编程的世界中的其他组件，开发者都需要仔细学习应该在什么时候怎么使用它们（_否则就很可能出错_）。讲解 RxJava 的朋友，应该考虑一下在什么时候以及以何种方式来介绍 `Subject`。我建议在介绍完常用的操作符之后，但是在介绍 `create()` 之前。

在这个系列中，我将详细介绍 `Subject`，包括它们的使用要求、实现结构，以及我们如何实现自定义的 `Subject`。

## 不确定的事件发射（Imperative eventing）

_译者注：标题属意译，Subject 解决的需求就是不确定什么时候可以发出事件。_

假设我们有一些数据希望通过 RxJava 来发出，但我们并不确定这些数据什么时候到来、以及有多少数据。显然 `just()` 和 `from()` 不能满足需求，但我们又不想用 `create()`，因为它会带来一些其他的问题。

最好的办法就是有一个对象既是 `Observable`，这样我们就可以去订阅它并进行一系列操作，它又是一个 `Observer`，这样我们就可以向它发出数据以及结束事件了。这种组合就是现在被称作 `Subject` 的类，在 Reactive-Streams 中叫做 `Processor`。

你也可以把它当做是实现多播的一种手段，我们无需担心 `Subscriber` 之间的线程安全性问题。

~~~ java
Subject changeEvents = ...
 
changeEvents.subscribe(System.out::println);
changeEvents.subscribe(System.out::println);
 
changeEvents.onNext("Added");
Thread.sleep(10);
changeEvents.onNext("Removed");
Thread.sleep(10);
changeEvents.onCompleted();
~~~

`Subject` 支持泛型，有时我们想要转发的就是接收到的数据类型，有时我们也希望转发完全不一样的类型。在 C# 中我们可以定义两种不同的类型：

~~~ java
interface ISubject<T>: IObserver<T>, IObservable<T> { }
 
interface ISubject<T, U>: IObserver<T>, IObservable<U> { }
~~~

但是 Java 泛型的类型擦除不允许我们像上面这样定义，所以它始终接受两个类型参数：

~~~ java
abstract class Subject<T, R>
extends Observable<R> implements Observer<T> { }
~~~

由于 RxJava 的 API 入口是一个类（_Observable_），为了保留组合的能力，Subject 继承自 Observable 并实现了 Observer。也许当时选择继承自 Subscriber 也是一个不错的选择，这样就能获得一些资源管理以及 backpressure 的能力，但是 Java 不允许多继承，而拥有组合的能力更重要（_所以继承了 Observable_）。

所以在上面的例子中，我们会有这样的代码：

~~~ java
Subject<String, String> changeEvents = ...
~~~

Subject 也是一种 hot Observable，没有 Subscriber 时也可以发射事件，就像广播电台一样，不会因为你关了收音机就停止广播，在收听的人就听到了，没在的人也无所谓。

在 RxJava 中这样的 Subject 叫 `PublishSubject`：

~~~ java
Subject<String, String> changeEvents = PublishSubject.create();
~~~

当然，`create()` 工厂方法可以免去用户在代码中重复类型参数，但为什么不能像 Rx.NET 那样直接调用 `new PublishSubject<>()` 呢？

主要原因是考虑到 `Observable` 类所定义的那套流利的 API。如果你还记得，我们在利用 `Observable.create()` 创建 Observable 时需要提供一个 `OnSubscribe`。在 Observable 内部会记住这个对象，并在每次有 Subscriber 订阅时，都调用 `OnSubscribe.call()`。

和其他 cold Observable 不一样，Subject 需要记录它的订阅者，这样所有的订阅者才能得到同样的数据，而这个记录需要同时在 OnSubscribe 和 Subject 中进行。不幸的是，Java 不允许在构造函数中的内部类访问外部类父类的成员，所以数据的共享必须在另一个单独的类中。而这就不是一件简单明了的事情了，所以构造 Subject 的过程就被封装在了像 `PublishSubject.create()` 这样的工厂方法中。（后面我们讲到 Subject 的构造时，我会结合实际例子进行详细讲解）

## 其他版本（Flavors）

有时候你不仅仅是想要发射出事件，你还希望可以考虑一下订阅者的情况。

假设你是一个电视内容发布商，你每周都会发布大量的内容。但是你的客户（_观众_）并不能一次性看完所有的内容，但他们也许想错过任何内容。所以智能电视和机顶盒自己就提供了一种缓存功能，它们会保存所有的内容，然后让用户按照自己的速度观看完所有的内容。

在程序的世界里，你可能希望发出的事件可以让不同的订阅者按照不同的进度接收到所有的事件，甚至是当你停止发射新事件之后，依然要可以收到所有的事件：姗姗来迟的订阅者依然要接收到之前积累的所有事件。

这种 Subject 叫 `ReplaySubject`。

默认情况下，我们创建的 `ReplaySubject` 拥有一个无限的缓冲区，用来保存已经产生的事件，并且会对每个 Subscriber 重放它们，包括结束事件。

但有时我们可能希望限制事件保留的时间，或者保留事件的数量，这样就不会每个 Subscriber 都从最开始接收数据了。RxJava 提供了不同的 API 来满足我们这样的需求：

+ `createWithSize(n)` 只会保留最近的 `n` 个元素。
+ `createWithTime(t, u)` 只会保留最近 `t` 时间内的数据。
+ `createWithTimeAndSize(n, t, u)` 只会保留最近 `t` 时间内的数据，且不超过 `n` 个。

这些差不多就够用了，但也有一些特殊情况下，我们还是需要自定义的 Subject。

例如我们有一个异步的计算任务，它会发出一个数据，然后结束事件流。`ReplaySubject` 能满足需求，但它太重了，而且它的开销对于这样一个简单的需求来说可能是不可接受的。RxJava 提供了一个叫 `AsyncSubject` 的类，它会记录收到的最后一个数据，然后当收到 `onCompleted` 时，所有已经订阅以及将来订阅的 Subscriber，都会立即收到这最后一个数据，并且紧接着有一个 `onCompleted`。和 ReplaySubject 不同，如果我们给 AsyncSubject 发送了一个 `onError` 事件，那之前收到的数据都会被忽略，所有的 Subscriber 都只会收到 onError。

RxJava 提供的最后一种 Subject 叫 `BehaviorSubject`，它只会重放最后的一个数据。当然，容量为 1 的 ReplaySubject 也可以提供这个功能，但和 ReplaySubject 不一样，BehaviorSubject 收到 onError/onCompleted 之后，保存的数据就会被丢弃了，后来的 Subscriber 只会收到 onError/onCompleted 了。创建 BehaviorSubject 时，我们可以提供一个初始值，也可以不提供。

## 一个响应式的列表

好了，经过了上面大段的干巴巴的文字只会，让我们来看看代码，看看 Subject 到底怎么使用。假设我们想要构建一个响应式的列表，当列表内容发生变化时，它能发出通知。我们希望在增加数据、删除数据以及数据内容更新时发出通知，并且：

+ 有一条通道用来通知变化的类型。
+ 有一条通道来通知是哪个元素导致的变化。
+ 以及一条通道来发布最近被加入到列表中的 10 个元素。

让我们把这个列表叫做 `ReactiveList`：

~~~ java
public final class ReactiveList<T> {
     
    public enum ChangeType { 
        ADD, REMOVE, UPDATE                      // (1)
    };
     
    final List<T> list = new ArrayList<>();      // (2)
     
    final PublishSubject<Changetype> changes = 
            PublishSubject.create();             // (3)
    final BehaviorSubject<T> changeValues = 
            BehaviorSubject.create();
    final ReplaySubject<T> latestAdded = 
            ReplaySubject.createWithSize(10);
     
    public Observable<ChangeType> changes() {    // (4)
        return changes;
    }
     
    public Observable<T> changeValues() {
        return changeValues;
    }
     
    public Observable<T> latestAdded() {
        return latestAdded;
    }
     
    public void add(T value) {
        // implement
    }
    public void remove(T value) {
        // implement
    }
    public void replace(T value, T newValue) {
        // implement
    }
}
~~~

它的结构中有几点值得一提：

1. 我们用枚举类型来定义变化的类型：增、删、改。
2. 实际的数据会保存在 `java.util.List` 中。
3. 每种通道都对应一个 Subject。
4. 并且我们提供了访问它们的方法。

现在让我们看看修改方法的实现：

~~~ java
    // ...
    public void add(T value) {
        list.add(value);
        changes.onNext(ChangeType.ADD);
        changeValues.onNext(value);
        latestAdded.onNext(value);
    }
    public void remove(T value) {
        if (list.remove(value)) {
            changes.onNext(ChangeType.REMOVE);
            changeValues.onNext(value);
        }
    }
    public void replace(T value, T newValue) {
        int index = list.indexOf(value);
        if (index >= 0) {
            list.set(index, newValue);
            changes.onNext(ChangeType.UPDATE);
            changeValues.onNext(newValue);
        }
    }
}
~~~

很简单，我们进行相应的修改操作，然后在相应的通道上发布事件。

## 一个更加响应式的列表

如果我们希望 `ReactiveList` 的输入（修改操作）也变得响应式，例如提供相应的 `Observer` 接口，让用户可以向其发出数据来更新我们的列表，应该怎么办？

为了简单起见，我去掉了 `replace()` 方法，但是加了一个 `list()` 方法，它会返回一个 Observable，发出当时的所有数据。

既然我们已经完全响应式了，我们就需要考虑对 Subject 的函数调用的串行化问题。Subject 实现了 Observer 接口，所以对它的 onXXX 方法的调用也不能并行发生。我们当然可以利用前面学到的串行访问实现方式，但由于这种需求大家都需要，所以 Subject 有一个 `toSerialized()` 操作符，它会保证 Subject 的串行访问。

~~~ java
public class MoreReactiveList<T> {
    public enum ChangeType { 
        ADD, REMOVE 
    };
     
    final List<T> list = new ArrayList<>();
     
    final Subject<ChangeType, ChangeType> changes;  // (1)
    final Subject<T, T> changeValues;
    final Observer<T> addObserver;                  // (2)
    final Observer<T> removeObserver;
     
    public Observable<ChangeType> changes() {
        return changes;
    }
     
    public Observable<T> changeValues() {
        return changeValues;
    }
     
    public Observable<T> list() {                   // (3)
        List<T> copy = new ArrayList<>();
        synchronized (list) {
            copy.addAll(list);
        }
        return Observable.from(copy);
    }
     
    public Observer<T> adder() {                    // (4)
        return addObserver;
    }
     
    public Observer<T> remover() {
        return removeObserver;
    }
 
    void onAdd(T value) {                           // (5)
        synchronized (list) {
            list.add(value);
        }
        changes.onNext(ChangeType.ADD);
        changeValues.onNext(value);
    }
 
    void onRemove(T value) {
        synchronized (list) {
            if (!list.remove(value)) {
                return;
            }
        }
        changes.onNext(ChangeType.REMOVE);
        changeValues.onNext(value);
    }
 
    void clear() {
        synchronized (list) {
            list.clear();
        }
    }
 
    public MoreReactiveList() {
        // implement
    }
}
~~~

代码结构发生了一些变化：

1. 由于我们需要保证串行访问，我们不能声明为各种具体的 Subject 类型了，我们需要声明为 `Subject<T, T>`。
2. 对增加、删除操作，我们都有一个 Observer。
3. 我们的 `list()` 方法会返回当前列表内容的快照（用 Observable 的形式）。注意对 `list` 访问的同步控制，因为我们会并发操作 list 的内容，所以需要进行同步以保证线程安全性。
4. 我们定义相关方法暴露出增删操作的 Observer。
5. 实际的增删操作被转移到了包私有的 `onAdd` 和 `onRemove` 中。

最后我们看看构造函数 `MoreReactiveList()` 的逻辑：

~~~ java
    // ...
    public MoreReactiveList() {
        changes = 
                PublishSubject.<ChangeType>create()
                .toSerialized();                     // (1)
 
        changeValues = 
                BehaviorSubject.<T>create()
                .toSerialized();                     
         
        addObserver = new SerializedObserver<>(      // (2)
            Observers.create(
                this::onAdd,
                t -> {
                    clear();
                    changes.onError(t);
                    changeValues.onError(t);
                },
                () -> { 
                    clear();
                    changes.onCompleted();
                    changeValues.onCompleted();
                }
        ));
        removeObserver = new SerializedObserver<>(   // (3)
            Observers.create(
                this::onRemove,
                t -> {
                    clear();
                    changes.onError(t);
                    changeValues.onError(t);
                },
                () -> { 
                    clear();
                    changes.onCompleted();
                    changeValues.onCompleted();
                }
        ));
    }
}
~~~

它的工作原理如下：

1. 和 `ReactiveList` 类似，我们需要设置输出的通道，但这次我们需要保证串行访问。
2. `addObserver` 是一个串行访问的 Observer，它在 onNext 中执行 `onAdd`，在 onError/onCompleted 中转发终结事件。
3. `removeObserver` 和 `addObserver` 类似，只不过 onNext 执行的是 `onRemove`。

让我们尝试一下：

~~~ java
MoreReactiveList<Long> list = new MoreReactiveList<>();
 
Observable.timer(0, 1, TimeUnit.SECONDS)
    .take(10)
    .subscribe(list.adder());
 
Observable.timer(4, 1, TimeUnit.SECONDS)
    .take(10)
    .subscribe(list.remover());
 
list.changes().subscribe(System.out::println);
 
list.changes()
.flatMap(e -> list.list().toList())
.subscribe(System.out::println);
 
list.changeValues.toBlocking().forEach(System.out::println);
~~~

在这个例子中，我们创建了一个响应式列表，以及两个 timber 来进行增加和删除操作。然后我们打印了变化的类型、每次变化时列表的内容，最后利用一个阻塞的订阅来等待终结事件。

## 总结

在本文中，我介绍了 Subject 的概念，以及 RxJava 中的不同实现，最后实现了两个类型来利用这些 Subject 的特性。

在下一篇文章中，我将讲讲实现自定义 Subject 的要求、结构以及算法。
