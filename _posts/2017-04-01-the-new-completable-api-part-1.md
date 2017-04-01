---
layout: post
title: 全新 Completable API（一）
tags:
    - Completable
---

原文 [The new Completable API (part 1)](http://akarnokd.blogspot.com/2015/12/the-new-completable-api-part-1.html){:target="_blank"}。

## 介绍

如果大家关注 GitHub 上 RxJava 的动态，那可能已经注意到有一个关于 `rx.Completable` 类的 PR。这个 PR 已经被合入了 1.x 分支（被标记为 `@Experimental`），并极有可能成为 1.1.1 版本的一部分。

在接下来的两篇文章中，我将首先介绍 Completable 这个类的用法，以及和已有的 Observable/Single 类的关系，然后介绍其内部的实现原理。

正如 `@Experimental` 注解所表明的，它的函数名字以及可见性在 1.1.1 发布之前都可能发生变化。

## Completable 类是什么？

我们可以把 Completable 看做是 Observable 功能的子集，它只可能有 onCompleted 和 onError 事件；它看起来像是 `Observable.empty()`，但和 empty 不同的是，它是一个活跃的类。Completable 在被订阅时会做一些有副作用的操作。通常 Completable 都包含一些延迟的计算，它只会通知这些计算成功与否。

和 Single 一样，Completable 也可以通过继承 `Observable<?>` 来实现，但很多 API 的设计者认为，把它的功能编写在一个单独的类里面，比用带上通配符的 Observable 子类来实现更具有表现力。

Completable 并不会发出任何数据，所以不需要考虑 backpressure，这样可以简化它内部的结构，但优化它的内部实现则需要更多非阻塞的原子操作的知识。

## Hello World!

让我们看看如何编写一个 Hello World 级别的 Completable：

~~~ java
Completable.fromAction(() -> System.out.println("Hello World!"))
.subscribe();
~~~

非常直观，我们有一系列的 fromXXX 函数，来接收不同的操作源：`Action`，`Callable`，`Single` 甚至是 `Observable`（当然后三者产生的数据都会被忽略掉）。在订阅的时候，我们和订阅普通的 Observable 一样，可以不提供参数，提供一个接收结束事件的 lambda 表达式，`rx.Subscriber` 或者专为 Completable 打造的订阅者 `rx.Completable.CompletableSubscriber`。

## 响应式的空数据流？

`CompletableSubscriber` 的定义和 Reactive-Stream 的 `Subscriber` 看起来很类似，重新设计这样一个接口而不是直接使用 `rx.Subscriber` 主要是从性能方面进行考虑：

~~~ java
public interface CompletableSubscriber {
 
    void onCompleted();
 
    void onError(Throwable e);
 
    void onSubscribe(Subscription d);
}
~~~

它有寻常的 `onCompleted()` 和  `onError()` 函数，但它并没有继承自 `Subscription`，也没有 `rx.Subscriber` 的 `add()` 函数，它通过 `onSubscribe` 函数接收一个 `Subscription`，以便可以取消订阅，就像 Reactive-Stream 那样。这样的设计有以下好处：

+ 它允许不同的 CompletableSubscriber 实现者决定是否暴露取消订阅的功能，这和 `rx.Subscriber` 不一样，后者任何人都可以取消订阅；
+ 在 `rx.Subscriber` 中，我们一定有一个 `SubscriptionList` 容器（可能会共享），用以支持资源管理。然而很多 Observable 的操作符自己并不使用（需要）资源管理功能，这样就会白白增加内存分配的开销；

终止事件的语义也和 Reactive-Stream 一致，当 onError 或者 onCompleted 被调用之后，之前收到的 Subscription 对象就应该被认为已经取消订阅了。

所以，它的协议看起来是这样的：

``` regex
onSubscribe (onError | onCompleted)?
```

它一定会有一次接收非空参数的 `onSubscribe` 调用，然后是一次接收非空 Throwable 参数的 onError 调用，或者一次 onCompleted 调用。和 Reactive-Stream 一样，在函数内部不能抛出任何 checked exception，以及除了 NullPointerException 之外的 unchecked exception。这并不是说这些函数里面不能发生异常，只是说异常应该抛往下游。但一定存在一些情况，我们没有机会把异常抛往下游（例如调用 onCompleted 之后发生异常），这时最后的选择就是捕获异常，并把异常交给

``` java
RxJavaPlugins.getInstance().getErrorHandler().handleError(e)
```

## Create，Lift 以及 Transform 操作

Completable 有三个辅助接口，有了 RxJava 的基本类型之后，也就变得简单了。

第一个接口定义了延迟计算的一种方式，并把结束事件发给 CompletableSubscriber：

~~~ java
public interface CompletableOnSubscribe
    extends Action1<CompletableSubscriber> { }
 
CompletableOnSubscribe complete = cs -> {
    cs.onSubscribe(Subscriptions.unsubscribed());
    cs.onCompleted();
}
~~~

它就是对接收 CompletableSubscriber 的 Action1 的一个别名。通过 lambda 表达式创建一个 CompletableOnSubscribe 的代码也很直观（但不要忘记在调用 onXXX 之前调用 onSubscribe 函数）。

第二个接口允许我们通过指定 CompletableSubscriber 层面的转换函数来完成 Completable 序列的应用：

~~~ java
public interface CompletableOperator 
    extends Func1<CompletableSubscriber, CompletableSubscriber> { }
 
CompletableOperator swap = child -> new CompletableSubscriber() {
    @Override
    public void onSubscribe(Subscription s) {
        child.onSubscribe(s);
    }
    @Override
    public void onCompleted() {
        child.onError(new RuntimeException());
    }
    @Override
    public void onError(Throwable e) {
        child.onCompleted();
    }
};
~~~

CompletableOperator 是 Func1 的一个别名，它允许我们交换、替换以及增强下游 CompletableSubscriber 的功能。示例代码演示了如何把两种终止事件互相交换。

最后一个接口允许我们把 Completable 操作的链条进行链接：

~~~ java
public interface CompletableTransformer
    extends Func1<Completable, Completable> { }
 
CompletableTransformer schedule = c -> 
    c.subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread());
~~~

有了这个接口，我们就可以通过 compose 函数把后来准备好的操作符链条接入已有的链条中了。示例代码演示了如何进行线程切换。

## 进入 Completable 的世界

_接下来主要是各个接口的介绍，就不一一翻译了，直接给出原文。_


和 Observable 一样，Completable 也有一系列的工厂方法，允许我们从不同的源头创建 Completable 流：

+ `create(CompletableOnSubscribe)`: let's you write your custom deferred computation that receives a CompletableSubscriber as its observer. The CompletableOnSubscribe is invoked for all CompletableSubscriber separately.
+ `complete()`: returns a constant instance of a Completable which calls onCompleted without doing anything else.
+ `defer(Func0<Completable>)`: calls the function for each incoming CompletableSubscriber which should create the actual Completable instance said subscriber will subscribe to.
+ `error(Throwable)`: it will emit the given constant Throwable to the incoming CompletableSubscribers.
+ `error(Func0<Throwable>)`: for each incoming CompletableSubscriber, the Func0 is called individually and the returned Throwable is emitted through onError.
+ `fromAction(Action0)`: let's you execute an action for each CompletableSubscriber which call onCompleted (or onError if the action throws an unchecked exception).
+ `fromCallable(Callable)`: unfortunately, Java doesn't have a standard interface for an action which returns void and can throw a checked exception (not even in 8). The closest thing is the Callable interface. This let's you write an action that doesn't require you to wrap the computation into a try-catch but mandates the return of some arbitrary value (ignored). Returning null is acceptable here.
+ `fromFuture(Future)`: let's you attach to a future and wait for its completion, literally. This blocks the subscriber's thread so you will have to use subscribeOn().
+ `fromObservable(Observable)`: let's you skip all values of the source and just react to its terminal events. The Observable is observed in an unbounded backpressure mode and the unsubscription (naturally) composes through.
+ `fromSingle(Single)`: let's you turn the onSuccess call into onCompleted call coming from the Single.
+ `never()`: does nothing other than setting an empty Subscription via onSubscribe.
+ `timer(long, TimeUnit)`: completes after the specified time elapsed.

除此之外，Observable 和 Single 都有一个 `toCompletable()` 函数。

fromXXX 的命名是故意设计为不一样的：Java 8 编译器对函数式接口重载的处理很容易出现歧义，所以名字就起得不一样，使用代码写起来会更方便。


## Leaving the Completable world

One has to, eventually, leave the Completable world and observe the terminal event in some fashion. The Completable offers some familiar methods to make this happen: subscribe(...).

We can group the subscribe() overloads into two sets. The first set returns a Subscription that allows external cancellation and the second relies on the provided class to allow/manage unsubscriptions.

The first group consists of the lambda-form subscriptions:

+ `subscribe()`: runs the Completable and relay any onError call to the RxJavaPlugins.
+ `subscribe(Action0)`: runs the Completable and calls the given Action0 on successful completion. The onError calls are still relayed to RxJavaPlugins.
+ `subscribe(Action1, Action0)`: runs the Completable and calls Action1 if it ended with an onError or calls Action0 if it ended with a normal onCompleted.

Since the lambda callbacks don't have access to the underlying Subscription sent through onSubscribe, these methods return a Subscription themselves to allow external unsubscription to happen. Without it, there wouldn't be any way of cancelling such subscriptions.


The second group of subscribe methods take the multi-method Subscriber instances:

+ `subscribe(CompletableSubscriber)`: runs the Completable and calls the appropriate onXXX methods on the supplied CompletableSubscriber instance.
+ `subscribe(Subscriber<T>)`: runs the Completable and calls the appropriate onXXX methods on the supplied rx.Subscriber instance.


Sometimes, one wants to wait for the completion on the current thread. Observable has a set of methods accessible through toBlocking() for this purpose. Since there are not many ways one can await the result of a Completable, the blocking methods are part of the Completable class itself:

+ `await()`: await the termination of the Completable indefinitely and rethrow any exception it received (wrapped into a RuntimeException if necessary).
+ `await(long, TimeUnit)`: same as await() but with bounded wait time which after a TimeoutException is thrown.
+ `get()`: await the termination of the Completable indefinitely, return null for successful completion or return the Throwable received via onError.
+ `get(long, TimeUnit)`: same as get() but with bounded wait time which after a TimeoutException is thrown.


## Completable operators

Finally, let's see what operators are available to work with an Completable. Unsurprisingly, many of them match their counterpart in Observable, however, a lot of them is missing because they don't make sense in a valueless stream. This include the familiar map, take, skip, flatMap, concatMap, switchMap, etc. operators.

The first set of operators is accessible as a static method and usually deal with a set of Completables. Many of them have overloads for varargs and Iterable sequences.

+ `amb(Completable...)`: terminates as soon as any of the source Completables terminates, cancelling the rest.
+ concat(Completable...)`: runs the Completable one after another until all complete successfully or one fails.
+ `merge(Completable...)`: runs the Completable instances "in parallel" and completes once all of them completed or any of them failed (cancelling the rest).
+ `mergeDelayError(Completable...)`: runs all Completable instances "in parallel" and terminates once all of them terminate; if all went successful, it terminates with onCompleted, otherwise, the failure Throwables are collected and emitted in onError.
+ `using(Func0, Func1, Action1)`: opens, uses and closes a resource for the duration of the Completable returned by Func1.

The second set of operators are the usual (valueless) transformations:

+ `ambWith(Completable)`: completes once either this or the other Completable terminates, cancelling the still running Completable.
+ `concatWith(Completable)`: runs the current and the other Completable in sequence.
+ `delay(long, TimeUnit)`: delays the delivery of the terminal events by a given time amount.
+ `endWith(...)`: continues the execution with another Completable, Single or Observable.
+ `lift(CompletableOperator)`: lifts a custom operator into the sequence that allows manipulationg the incoming downstream's CompletableSubscriber's lifecycle and event delivery in some manner before continuing the subscribing upstream.
+ `mergeWith(Completable)`: completes once both this and the other Completable complete normally
+ `observeOn(Scheduler)`: moves the observation of the terminal events (or just onCompletded) to the specified Scheduler.
+ `onErrorComplete()`: If this Completable terminates with an onError, the exception is dropped and downstream receives just onCompleted.
+ `onErrorComplete(Func1)`: The supplied predicate will receive the exception and should return true if the exception should be dropped and replaced by a onCompleted event.
+ `onErrorResumeNext(Func1)`: If this Completable fails, the supplied function will receive the exception and it should return another Completable to resume with.
+ `repeat()`: repeatedly executes this Completable (or a number of times in another overload)
+ `repeatWhen(Func1)`: repeatedly execute this Completable if the Observable returned by the function emits a value or terminate if this Observable emits a terminal event.
+ `retry()`: retries this Completable if it failed indefinitely (or after checking some condition in other overloads):
+ `retryWhen(Func1)`: retries this Completable if it failed and the Observable returned by the function emits a value in response to the current exception or terminates if this Observable emits a terminal event.
+ `startWith(...)`: begins the execution with the given Completable, Single or Observable and resumes with the current Completable.
+ `timeout(long, TimeUnit, Completable)`: switches to another Completable if this completable doesn't terminate within the specified time window.
+ `to(Func1)`: allows fluent conversion by calling a function with this Completable instance and returning the result.
+ `toObservable()`: converts this Completable into an empty Observable that terminates if this Completable terminates.
+ `toSingle(Func0<T>)`: converts this Completable into a Single in a way that when the Completable completes normally, the value provided by the Func0 is emitted as onSuccess while an onError just passes through.
+ `toSingleDefault(T)`: converts this Completable into a Single in a way that when the Completable completes normally, the value provided is emitted as onSuccess while an onError just passes through.
+ `unsubscribeOn(Scheduler)`: when the downstream calls unsubscribe on the supplied Subscription via onSubscribe, the action will be executed on the specified scheduler (and will propagate upstream).

The final set of operators support executing callbacks at various lifecycle stages (which can be used for debugging or other similar side-effecting purposes):

+ `doAfterTerminate(Action0)`: executes the action after the terminal event has been sent downstream CompletableSubscriber.
+ `doOnComplete(Action0)`: executes an action just before the completion event is sent downstream.
+ `doOnError(Action1)`: calls the action with the exception in a failed Completable just before the error is sent downstream.
+ `doOnTerminate(Action0)`: executes the action just before any terminal event is sent downstream.
+ `doOnSubscribe(Action1)`: calls the action with the Subscription instance received during the subscription phase.
+ `doOnUnsubscribe(Action0)`: executes the action if the downstream unsubscribed the Subscription connecting the stages.
+ `doOnLifecycle(...)`: combines the previous operators into a single operator and calls the appropriate action.

Currently, there are no equivalent Subject implementations nor publish/replay/cache methods available. Depending on the need for these, they can be added later on. Note however that since Completable deals only with terminal events, all Observable-based Subject implementation have just a single equivalent, Completable-based Subject implementation and there is only one way to implement the publish/replay/cache methods.

It is likely the existing Completable operators can be extended or other existing Observable operators matched. Until then, you can use the

toObservable().operator.toCompletabe()

conversion pattern to reach out to these unavailable operators. In addition, I didn't list all overloads so please consult with the source code of the class (or the Javadoc once it becomes available online).

## 总结

在本文中，我介绍了新的 Completable 基本类型，以及现在已有的所有工厂方法以及操作符。它的使用方式和 Observable 以及 Single 非常类似，唯一的区别就是它不用处理数据，只需要处理终止事件，因此很多操作符都没有意义（_所以没有这些操作符_）。

在下一篇文章中，我将讲解如何实现 CompletableOnSubscribe 和 CompletableOperator 接口。
