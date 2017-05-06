---
layout: post
title: 如何编写自定义响应式基础类型
tags:
---

原文 [Writing a custom reactive base type](http://akarnokd.blogspot.com/2016/03/writing-custom-reactive-base-type.html){:target="_blank"}。

## 介绍

一直以来，大家都在问如何实现自己的响应式类型。尽管 RxJava 的 `Observable` 有大量方法，也允许通过 `lift()`、`extend()` 以及 `compose()` 进行扩展，大家仍会希望 Observable 拥有某个 `xyz()` 操作符，或者在某个调用链中不允许调用 `uvw()`。

第一个情况，其实是希望在吃透整个 RxJava 项目之前就能增加自定义的操作符函数，而这个需求其实和 JVM 环境中的响应式编程同样古老。当我最初把 Rx.NET 移植到 Java 中时，我也面临了同样的问题，因为 .NET 早在 2010 年就支持了方法扩展。而 Java 并不支持方法扩展，而且这一特性的提议也在 Java 8 开发时期被否决，他们选择了默认方法这一特性，理由是扩展方法无法被重载。的确，扩展方法无法被重载，但它可以被另一个类的另一个方法替代。

第二个情况，我们希望隐藏或者移除某些操作符，因为在我们自定义的 Observable 类型中，有些操作符没有意义。例如我们有一个 `ParallelObservable`，它把输入序列在内部分为并行处理的流水线，那么 `map()` 和 `filter()` 就是有意义的，而 `take()` 和 `skip()` 则没有意义。

## 包装（Wrapping）

上面的两种情况都可以通过包装 Observable 类来实现：

~~~ java
public final class MyObservable<T> {
    private Observable<T> actual;
    public MyObservable<T>(Observable<T> actual) {
        this.actual = actual;
    }
}
~~~

然后我们就可以通过转发来增加我们的操作符了：

~~~ java
    // ...
    public static <T> MyObservable<T> create(Observable<T> o) {
        return new MyObservable<T>(o);
    }

    public static <T> MyObservable<T> just(T value) {
        return create(Observable.just(value));
    }

    public final MyObservable<T> goAsync() {
        return create(actual.subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread()));
    }

    public final <R> MyObservable<R> map(Func1<T, R> mapper) {
        return create(actual.map(mapper));
    }

    public final void subscribe(Subscriber<? super T> subscriber) {
        actual.subscribe(subscriber);
    }
    // ...
~~~

这样我们就实现了上述两个目标：去掉了不必要的操作符，也引入了新的操作符。

如果我们看 RxJava 的源码，就会发现同样的模式，实际的对象都是 `OnSubscribe` 或者 `Publisher` 类型，而 `Observable` 则通过各种操作符极大地丰富了这些类型的功能。

## 互操作（Interoperation）

`MyObservable` 看起来已经足够了，但最终我们会需要和普通的 `Observable` 或者是其他人实现的 `YourObservable` 互相操作。由于它们是不同的类型，我们需要一个公共的父类来使它们可以交互。最自然的想法当然是都实现一个 `toObservable()` 函数，返回内部的 `Observable`，但这导致我们必须调用这个额外的方法。相反，我们可以让每个自定义类型都继承自同一个基类，或者实现同一个接口，它们包含互相操作所需的最小方法集合。

在 RxJava 1.x 中，显而易见的选项是 `Observable`，但它并不够好，因为它的方法都是 final 的，无法被被重写，而且这些方法都返回 `Observable` 而不是 `MyObservable`。不幸的是，RxJava 1.x 在这一点上无能为力，因为它要保证二进制兼容性。

幸运的是，在 RxJava 2.x 中，最基本的类型并不是 `Observable`（`Flowable`），而是 `Publisher`。所有的 `Observable` 都是 `Publisher`，而很多操作符都是接受 `Publisher` 参数。这样的好处就是我们可以兼容 RxJava 2.x 之外的基于 `Publisher` 的类型。之所以可以做到这样，是因为操作符只需要它的上游有一个 `subscribe(Subscriber<? super T> s)` 函数即可。

因此，如果我们的目标是 2.x，那么 `MyObservable` 就应该实现 `Publisher` 接口，这样它就立即可以和其他正经的响应式编程库兼容了：

~~~ java
public class MyObservable<T> implements Publisher<T> {
    // ...
    @Override
    public void subscribe(Subscriber<? super T> subscriber) {
        actual.subscribe(subscriber);
    }
    // ...
}
~~~

## 扩展（Extension）

有了这个 `MyObservable` 之后，我们可能需要不同的响应式类型以应对不同的使用场景，但复制所有的操作符就太繁琐了。第一想法当然是把 `MyObservable` 作为其他响应式类型的基类，但这同样会遇到 `Observable` -> `MyObservable` 的类型：操作符返回的类型不一样。

我相信 Java 8 的 Streams API 也遇到了同样的问题，如果我们看看签名：`Stream extends BaseStream<T, Stream<T>>` 和 `BaseStream<T, S extends BaseStream<T, S>>`，就会发现非常别扭，父类的类型参数居然需要子类型。这样做是为了能在方法签名中捕获子类型，这样如果我们创建了一个 `MyStream` 类型，那所有的方法的返回类型都将是 `MyStream`。

我们也可以用类似的方式定义 `MyObservable` 类型：

~~~ java
public class MyObservable<T, S extends MyObservable<T, S>> implements Publisher<T> {
    
    final Publisher<? extends T> actual;
    
    public MyObservable(Publisher<? extends T> actual) {
        this.actual = actual;
    }
    
    @SuppressWarnings("unchecked")
    public <R, U extends MyObservable<R, U>> U wrap(Publisher<? extends R> my) {
        return (U)new MyObservable<R, S>(my);
    }
    
    public final <R, U extends MyObservable<R, U>> U map(Function<? super T, ? extends R> mapper) {
        return wrap(Flowable.fromPublisher(actual).map(mapper));
    }
    
    @Override
    public void subscribe(Subscriber<? super T> s) {
        actual.subscribe(s);
    }
}
~~~

一大堆泛型的东西，我们编写了一个 `wrap()` 函数，用于把某种 `Publisher` 类型包装为 `MyObservable` 类型，并且在 `map()` 函数中调用了 `wrap()` 以确保正确的结果类型。`MyObservable` 的子类则要重写 wrap 函数以提供它们自己的类型：

~~~ java
public class TheirObservable<T> extends MyObservable<T, TheirObservable<T>> {
    public TheirObservable(Publisher<? extends T> actual) {
        super(actual);
    }

    @SuppressWarnings("unchecked")
    @Override
    public <R, U extends MyObservable<R, U>> U wrap(Publisher<? extends R> my) {
        return (U) new TheirObservable<R>(my);
    }
}
~~~

让我们试一下：

~~~ java
public static void main(String[] args) {
    TheirObservable<Integer> their = new TheirObservable<>(Flowable.just(1));
    
    TheirObservable<String> out = their.map(v -> v.toString());

    Flowable.fromPublisher(out).subscribe(System.out::println);
}
~~~

结果符合预期，没有编译错误，也能打印出 1。

现在让我们为 `TheirObservable` 增加一个 `take()` 操作符：

~~~ java
    @SuppressWarnings({ "rawtypes", "unchecked" })
    public <U extends TheirObservable<T>> U take(int n) {
        Flowable<T> p = Flowable.fromPublisher(actual);
        Flowable<T> u = p.take(n);
        return (U)(TheirObservable)wrap(u);
    }
~~~

_译者注：原作者这里贴了 filter 的代码，我按照自己的理解写了 take 的代码。_

函数签名变得越来越复杂了，类型系统正在反击！我们需要使用裸类型和强转来使得结果看起来像是目标类型。此外，如果我们编写 `their.map(v -> v.toString()).take(1);` 这样的代码，编译器会提示找不到 `take()` 函数。因为 `map()` 只有在我们把返回值赋值给 `MyObservable` 类型时，它才会返回 `MyObservable` 类型。为了让类型推导正确工作，我们不得不把链式调用拆分为单独的调用：

~~~ java
    TheirObservable<Integer> their2 = new TheirObservable<>(Flowable.just(1));
    TheirObservable<String> step1 = their2.map(v -> v.toString());
    TheirObservable<String> step2 = step1.take(1);
    Flowable.fromPublisher(step2).subscribe(System.out::println);
~~~

接下来让我们把 `TheirObservable` 扩展为 `AllObservable`，然后增加 `filter()` 方法：

~~~ java
public static class AllObservable<T> extends TheirObservable<T> {
    public AllObservable(Publisher<? extends T> actual) {
        super(actual);
    }

    @Override
    <R, U extends MyObservable<R, U>> U wrap(Publisher<? extends R> my) {
        return (U)new AllObservable<R>(my);
    }
    
    @SuppressWarnings({ "rawtypes", "unchecked" })
    public <U extends AllObservable<T>> U filter(Predicate<? super T> predicate) {
        Flowable<T> p = Flowable.fromPublisher(actual);
        Flowable<T> u = p.filter(predicate);
        return (U)(AllObservable)wrap(u);
    }
}
~~~

然后使用它：

~~~ java
    AllObservable<Integer> all = new AllObservable<>(Flowable.just(1));
    
    AllObservable<String> step1 = all.map(v -> v.toString());

    AllObservable<String> step2 = step1.take(1);
    
    AllObservable<String> step3 = step2.filter(v -> true);

    Flowable.fromPublisher(step3).subscribe(System.out::println);
~~~

不幸的是，上面的代码无法编译，因为 `map()` 不是返回的 `AllObservable`，也就是说 `AllObservable` 不是 `MyObservable<String, U extends MyObservable<String, U>>`。把 `step1` 改成 `TheirObservable<String>` 可以解决编译的问题。然而，如果我们想要交换 `filter()` 和 `take()` 的顺序，`step1` 就不再是 `AllObservable` 了，而 `filter()` 也就无法调用了。

## 总结

我们能解决 `AllObservable` 的问题吗？我不知道，就我对 Java 类型系统和类型接口的理解，现在只能做到这个程度了。

RxJava 2.x 会有这样的结构吗？如果由我来定，那肯定就是不会了。为了支持这样的结构，我们每次都需要包装，而我一直想摆脱所有的 `lift()` 和 `create()`，而且这样做也会让类型和函数的签名更加复杂。

因此，如果有人希望这样做，上面的例子就展示了如何在不修改 RxJava 的情况下进行包装，并且可以自己控制暴露哪些 API。这也算是“组合优于继承”的一个很好的例子。
