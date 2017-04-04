---
layout: post
title: 全新 Completable API（二）
tags:
    - Completable
---

原文 [The new Completable API (part 2)](http://akarnokd.blogspot.com/2015/12/the-new-completable-api-part-2-final.html){:target="_blank"}。

## 介绍

在本文中，我将展示如何实现 Completable 的操作符（包括数据源和转换操作符）。由于 Completable API 并不会发出数据，只有 onError 和 onCompleted 事件，所以它的操作符比 Observable 少得多。

## Empty

我们首先实现 empty() 操作符，它会在被订阅时立即发出 onCompleted 事件。我们利用 CompletableOnSubscribe 来实现这一操作符：

~~~ java
Completable empty = Completable.create(completableSubscriber -> {
    BooleanSubscription cancel = new BooleanSubscription();    // (1)
     
    completableSubscriber.onSubscribe(cancel);                 // (2)
     
    if (!cancel.isUnsubscribed()) {
        cancel.unsubscribe();                                  // (3)
        completableSubscriber.onCompleted();                   // (4)
    }
});
~~~

这里我们无需引入类型参数。Completable 的 API 设计遵循了 Reactive-Stream 规范，它通过调用 onSubscribe 传递一个 Subscription 给下游，使得下游可以取消上游。

我们需要先创建一个 Subscription 实例，这里我们用了一个 BooleanSubscription 用于检查下游是否已经取消订阅（1）。在发出终止事件之前，我们需要先调用 onSubscribe，并把 BooleanSubscription 对象传递给下游（2）。这一步是必须的，如果漏掉这一步，可能大多数操作符都会抛出 NullPointerException，或者是在转换为 Observable 时取消订阅将失效。

和 Reactive-Stream 规范中定义的一样，终止事件发生后，我们要把 Subscription 等同于已被取消订阅，所以我们在调用 onCompleted 之前自行取消订阅（3）。

上面的代码看起来对一个简单如 empty 这样的操作符未免有些复杂，我们可以稍作简化：

~~~ java
Completable empty = Completable.create(completableSubscriber -> {
    completableSubscriber.onSubscribe(Subscriptions.unsubscribed());
    completableSubscriber.onCompleted();
});
~~~

这里我们直接给 onSubscribe 传递了一个已经取消订阅的 Subscription，然后立即调用 onCompleted。这么做有以下几个原因，由于取消订阅本身就只是一个尽力而为的保证，那就意味着：a) 我们必须做好准备，以应对取消订阅之后到来的事件；b) onSubscribe 和 onCompleted 之间的时间窗口真的很小，所以对于绝大部分异步场景，isUnsubscribed 都会返回 false；c) Subscription 在 onCompleted 调用之前也应该被认为是已经取消了。

## Empty delayed

假设我们需要在延迟一段时间之后发出 onCompleted 事件。为此我们需要一个 `Scheduler.Worker` 来进行延迟执行，但我们也得保证延迟执行的代码可以被取消。

~~~ java
public static Completable emptyDelayed(
        long delay, TimeUnit unit, Scheduler scheduler) {
    return Completable.create(cs -> {
        Scheduler.Worker w = scheduler.createWorker();
         
        cs.onSubscribe(w);
 
        w.schedule(() -> {
            try {
                cs.onCompleted();
            } finally {
                w.unsubscribe();
            }
        }, delay, unit);
    });
}
~~~

幸运的是，`Scheduler.Worker` 实现了 Subscription 接口，所以我们可以在调度延迟任务之前，直接把它传给 `CompletableSubscriber`。在调度的任务中，我们在调用 onCompleted 之后再取消订阅 worker，显然在调用 onCompleted 时 worker 并没有被取消订阅。首先即便 Reactive-Stream 规范规定，`org.reactivestreams.Subscription` 在终止事件到来时应该被认为已经取消订阅了，但现在我们之所以可以这样做，是因为我们无法检查 Subscription 是否已经被取消订阅。虽然在 RxJava 的实现中，我们可以检查是否已经取消订阅，但检查上游是否认为我们已经被取消订阅，并没有什么意义。其次之所以 onCompleted 和 unsubscribe 的顺序很重要，是因为如果先取消了 worker，我们在调用 onCompleted 时可能在下游被意外中断。

如果我们确实想要确保下游收到的 Subscription 已经被取消订阅，我们就得用 MultipleAssignmentSubscription 实现得没这么直观：

~~~ java
Scheduler.Worker w = scheduler.createWorker();
 
MultipleAssignmentSubscription mas = 
    new MultipleAssignmentSubscription();
 
mas.set(w);
 
cs.onSubscribe(mas);
 
w.schedule(() -> {
    mas.set(Subscriptions.unsubscribed());
    mas.unsubscribe();
  
    try {
        cs.onCompleted();
    } finally {
        w.unsubscribe();
    }
}, delay, unit);
~~~

我们不直接把 worker 传给下游，而是用 MultipleAssignmentSubscription 包装一层，并在调用 onCompleted 之前把它的内容替换为一个已经被取消订阅的 subscription，然后取消订阅整个容器类。使用 MultipleAssignmentSubscription 容器，以及替换 MultipleAssignmentSubscription 的内容，都是为了防止过早取消订阅 worker。SerialSubscription 和 CompositeSubscription 都无法达成目标。

最后，调度 onCompleted 的执行时需要格外小心，尤其是使用 RxJava 2.x 的 scheduler 时。RxJava 2.x 的 scheduler 允许“直接调度”，也就是说我们无需创建 worker 并在使用完毕之后取消订阅 worker。如果我们只需要调度一次任务，那这种形式可以减小开销，当然这种形式不保证同一个 scheduler 上调度的任务执行的先后顺序（这一点在这里可以接收）。所以 emptyDelayed 也可以这样实现：

~~~ java
cs.onSubscribe(
     scheduler.scheduleDirect(cs::onCompleted, delay, unit));
~~~

但是上面的代码存在竞争：有可能被调度的任务，以及任务中要发出的 onCompleted 事件，都会在 scheduleDirect 返回之前被执行（_例如 delay 是 0_），那这时 subscriber 在收到 onCompleted 之前就没有收到 onSubscribe 了。更坏的情况是 onError 和 onCompleted 可能同时发生，这违反了串行访问的原则。解决办法也是引入一个中间层：

~~~ java
MultipleAssignmentDisposable mad = 
    new MultipleAssignmentDisposable();
 
cs.onSubscribe(mad);
 
mad.set(
    scheduler.scheduleDirect(cs::onCompleted, delay, unit));
~~~

_注：由于命名冲突，RxJava 2.x 的资源管理类叫 Disposable 而不是 Subscription，我会很快发布一个关于 2.x 的系列文章。_

上面这个例子也预示了 Completable API 的一个特性：资源管理需要操作符自己负责，这里并没有像 `rx.Subscriber` 一样的 `add(Subscription)` 方法。这一点小小的不便需要我们在涉及到任务调度，或者涉及到多个资源管理时，要使用[Subscription 容器类](/AdvancedRxJava/2016/07/15/operator-concurrency-primitives-subscription-containers-1/)。这一设定的好处就是那些不需要资源管理的操作符，就不需要和 `rx.Subscriber` 一样相关的代码了，这样能节省内存分配，提高性能。

## First completed

在我小学的时候，老师们经常发起一些小的挑战，同学们谁第一个完成就能得到一份小奖励。这和 `amb()` 很类似：第一个结束的数据源是赢家。当然，第一个发生错误的数据源也是赢家，在现实生活中失败显然不会让我们胜出。

那如何为 Completable 实现一个 amb 操作符？显然，不像 `Observable.amb`，我们不需要保存胜出 Completable 的信息（因为它已经终止了），所以我们不需要[使用 trikery 的索引以及其他麻烦事](/AdvancedRxJava/2016/10/06/operator-internals-s-amb-ambwith/)，使用一个简单的 AtomicBoolean 就够了。

~~~ java
public static Completable amb(Completable... students) {
    return Completable.create(principal -> {
        AtomicBoolean done = new AtomicBoolean();            // (1)
 
        CompositeSubscription all = 
             new CompositeSubscription();                    // (2)
 
        CompletableSubscriber teacher = 
                new CompletableSubscriber() {
            @Override
            public void onSubscribe(Subscription s) {
                all.add(s);                                  // (3)
            }
             
            @Override
            public void onCompleted() {
                if (done.compareAndSet(false, true)) {       // (4)
                    all.unsubscribe();
                    principal.onCompleted();
                }
            }
 
            @Override
            public void onError(Throwable e) {
                if (done.compareAndSet(false, true)) {       // (5)
                    all.unsubscribe();
                    principal.onError(e);
                }
            }
        };
 
        principal.onSubscribe(all);                          // (6)
 
        for (Completable student : students) {
            if (done.get() || all.isUnsubscribed()) {        // (7)
                return;
            }
            student.subscribe(teacher);                      // (8)
        }
    });
}
~~~

在前面小学挑战的例子中，只会有一个老师检查所有学生的进展。这是一个针对 Completable 很有趣的优化（Reactive-Stream 里面也有），而且**技术上**它是切实可行的：teacher 这个 CompletableSubscriber 是无状态的，它只负责转发 onXXX 事件，而不需要记住是谁发出的这个事件。这里我强调**技术上**，因为 Reactive-Stream 规范禁止用同一个 Subscriber 订阅到多个 Publisher 上，但这一点在我看来是太过严格了：库的编写者，他们很清楚地知道自己在做什么，应该允许他们这样做。毫无意外的是，我们只需要把 teacher 的创建移到（8）处即可，其他的代码都不需要变化；显然，这也说明了 Subscriber 是可以被复用的。尽管如此，让我们看看上面值得关注的几个地方：

1. 我们创建了一个共享的 done 变量，它会在任何一个学生通知老师时（onCompleted 或者 onError）被设置为 true；
2. 如果校长不喜欢这个挑战，他可以随时取消整个挑战；
3. 老师需要在学生到来时进行注册；
4. 并且确保第一个学生发出终止事件时，也通知校长（不幸的是，校长无法知道是哪个学生胜出）。此时其他的学生也就没必要继续了，所以我们取消整个挑战；
5. 当然，如果发生了错误，处理方式也是一样的；
6. 我们允许校长直接取消整个挑战；
7. 我们把每挑战的材料交给每个学生（让老师订阅学生）。当然有可能我们还在订阅过程中挑战就已经结束了，那我们也就无需继续订阅了。

## When all completed

继续我们上小学的例子，有时候学生的表现需要被集体评估，整个活动需要等到所有学生都完成评估后才能结束。如果发生了意外，我们可能需要终止整个活动，并且把受伤的学生一起送到救护车。

我希望这样的设定听起来很熟悉，如果不熟悉，它们其实分别对应着 merge 和 mergeDelayError 操作符，具体是哪个取决于学校的政策。

让我们先看看最简单的情况，我们知道学生的数量，而且任何错误都会终止整个活动：

~~~ java
public static Completable merge(Completable... students) {
    return Completable.create(principal -> {
        AtomicInteger remaining = 
            new AtomicInteger(students.length);              // (1)
 
        CompositeSubscription all = 
            new CompositeSubscription();
 
        CompletableSubscriber evaluator = 
                 new CompletableSubscriber() {
             @Override
             public void onSubscribe(Subscription s) {
                 all.add(s);
             }
              
             @Override
             public void onCompleted() {
                 if (remaining.decrementAndGet() == 0) {     // (2)
                     all.unsubscribe();
                     principal.onCompleted();
                 }
             }
 
             @Override
             public void onError(Throwable e) {
                 if (remaining.getAndSet(0) > 0) {           // (3)
                     all.unsubscribe();
                     principal.onError(e);
                 }
             }
        };
 
        principal.onSubscribe(all);
         
        for (Completable student : students) {
            if (all.isUnsubscribed() 
                || remaining.get() <= 0) {                   // (4)
                return;
            }
            student.subscribe(evaluator);
        }
    });
}
~~~

上面的实现看起来和 `amb()` 比较类似，但也有以下几点不同：

1. 我们需要原子地计数（递减）成功完成评估的学生数量；
2. 一旦计数递减到 0，我们就通知校长评估完成；
3. 如果有学生发出了错误，那我们就原子地把计数置为 0，如果此前计数不为 0，我们就取消掉其他所有学生，并向校长通报错误。注意错误的通报最多只会发生一次，因为如果有多个 onError 并发到来，只会有一个成功地把计数置为 0。任何后续的结束事件都会继续递减计数器；
4. 如果发生了错误，或者活动被取消，我们就不再继续订阅了，以避免不必要的开销；

但如果我们不知道准确的学生数量，而且发生了错误之后，我们不希望结束整个活动，应该怎么做？这让我们的错误管理以及对学生的追踪变得复杂了一些：

~~~ java
public static Completable mergeDelayError(
        Iterable<? extends Completable> students) {
    return Completable.create(principal -> {
        AtomicInteger wip = new AtomicInteger(1);         // (1)
 
        CompositeSubscription all = 
            new CompositeSubscription();
 
        Queue<Throwable> errors = 
            new ConcurrentLinkedQueue<>();                // (2)
 
        CompletableSubscriber evaluator = 
                new CompletableSubscriber() {
            @Override
            public void onSubscribe(Subscription s) {
                all.add(s);
            }
 
            @Override
            public void onCompleted() {
                if (wip.decrementAndGet() == 0) {         // (3)
                    if (errors.isEmpty()) {
                        principal.onCompleted();
                    } else {
                        principal.onError(
                            new CompositeException(errors));
                    }
                }
            }
 
            @Override
            public void onError(Throwable e) {
                errors.offer(e);                          // (4)
                onCompleted();
            }
        };
 
        principal.onSubscribe(all);
 
        for (Completable student : students) {
            if (all.isUnsubscribed()) {                   // (5)
                return;
            }
            wip.getAndIncrement();                        // (6)
            student.subscribe(evaluator);
        }
 
        evaluator.onCompleted();                          // (7)
    });
}
~~~

同样，结构看起来很类似，但不同的要求需要用不同的算法来实现：

1. 首先我们的 wip 计数器初始值是 1。我们不知道会有多少个学生，但我们知道当 wip 变成 0 的时候，整个活动就结束了；
2. 我们会把所有的异常都保存到一个并发的队列中；
3. 终止条件在 evaluator 的 onCompleted 中检查：如果 wip 变成了 0，我们就检查异常队列是否为空，如果非空我们就把它们一起作为 CompositeException 发给校长；否则我们就告知校长活动成功结束；
4. 由于异常并不会提前终止活动，所以发送异常的处理和正常结束一样，但在此之前我们需要把异常加入到队列中（注意，行为不当的 Completable 会导致出错，如果它发出了多个终止事件，就会导致提前终止）；
5. 在订阅的循环中，我们只能检查校长是否需要取消活动，wip 的值在这里无法使用，因为在循环过程中，它至少是 1（_因为初始值是 1_）；
6. 对每一个学生，我们先递增 wip，再订阅它；这保证了我们在循环过程中，wip 至少是 1；
7. 最后我们手动调用一次 `evaluator.onCompleted()`，表示没有更多的 Completable 了，这使得 wip 可以被递减到 0 进而终止整个活动；

在 RxJava 1.x 的 Observable 体系中，很难出现代码如此紧凑，复用程度如此高的场景（甚至不可能出现）。

## Transformative operators

我不认为能有多少 Completable 的“序列”是可以被变换的。大多数 Observable 的操作符都没有意义，所以在 Completable 的 API 中省略了。但还是有几个例子可以看看的。

首先我们可能想要忽略错误事件，由于这里不涉及到 onNext，所以错误发生时我们最好的选择就是自行调用 onCompleted：

~~~ java
CompletableOperator onErrorComplete = cs -> {
    return new CompletableSubscriber() {
        @Override
        public void onSubscribe(Subscription s) {
            cs.onSubscribe(s);
        }
  
        @Override
        public void onCompleted() {
            cs.onCompleted();
        }
 
        @Override
        public void onError(Throwable e) {
            cs.onCompleted();
        }
    };
};
 
source.lift(onErrorComplete).subscribe();
~~~

当然，我们也可以在错误发生时订阅到另一个 Completable：

~~~ java
public static Completable onErrorResumeNext(
        Completable first,
        Func1<Throwable, ? extends Completable> otherFactory) {
 
    return first.lift(cs -> new CompletableSubscriber() {
        final SerialSubscription serial = 
            new SerialSubscription();                          // (1)
        boolean once;
 
        @Override
        public void onSubscribe(Subscription s) {
            serial.set(s);                                     // (2)
        }
 
        @Override
        public void onCompleted() {
            cs.onCompleted();
        }
 
        @Override
        public void onError(Throwable e) {
            if (!once) {                                       // (3)
                once = true;
                otherFactory.call(e).subscribe(this);          // (4)
            } else {
                cs.onError(e);                                 // (5)
            }
        }
    });
}
~~~

在上面的例子中，我们把 CompletableOperator lift 到传入的 Completable 上。在操作符主体中，我们返回一个具有以下行为的 CompletableSubscriber：

1. 由于我们可能会切换上游，所以我们需要把第一个 Completable 传入的 Subscription 更换为其他 Completable 传入的 Subscription。这里我们也可以使用 MultipleAssignmentSubscription。这和 Observable 操作符中常见的 ProducerArbiter 机制差不多，尽管简单了不少。此外，我们会在后来的 Completable 中复用这个 serial 对象，当然，如果它也出错了，我们就不会再尝试了；
2. 我们把 Subscription 设置到 SerialSubscription 中，把老的 Subscription 淘汰掉；
3. 如果 first 发生了错误，我们需要确保订阅到新的 Completable 只会发生一次；
4. 这里我们复用了 CompletableSubscriber 对象，节省了一次内存分配。（2）处的代码确保了取消链条被正确维护。这里为了简洁，我省略了工厂方法调用周围的 try-catch；如果发生了异常，我们可以把它和 onError 的异常合并为一个 CompositeException，并调用 `cs.onError`；
5. 如果第二个 Completable 也发生了异常，我们就把错误发往下游；

考虑一下，如果我们需要在 onCompleted 发生后执行类似的逻辑（例如 `andThen`，`endWith`，`concatWith`），我们可以用类似的方式实现。只不过我们不是在 onError 中做事，而是在 onCompleted 中做事：

~~~ java
// ...
@Override
public void onCompleted() {
    if (!once) {                                        // (3)
        once = true;
        otherFactory.call(e).subscribe(this);           // (4)
    } else {
        cs.onCompleted();                               // (5)
    }
}
 
@Override
public void onError(Throwable e) {
    cs.onError(e);
}
~~~

最后，我们可能需要在第一个 Completable 超时后，切换到另一个上：

~~~ java
public static Completable timeout(
        Completable first,
        long timeout, TimeUnit unit, Scheduler scheduler
        Completable other) {
     return first.lift(cs -> new CompletableSubscriber() {
 
         final CompositeSubscription csub = 
             new CompositeSubscription();                    // (1)
 
         final AtomicBoolean once = new AtomicBoolean();     // (2)
 
 
         @Override
         public void onSubscribe(Subscription s) {
             csub.add(s);                                    // (3)
 
             Scheduler.Worker w = scheduler.createWorker();
             csub.add(w);
 
             cs.onSubscribe(csub);
 
             w.schedule(this::onTimeout, timeout, unit);     // (4)
         }
 
         @Override
         public void onCompleted() {
             if (once.compareAndSet(false, true)) {
                 csub.unsubscribe();                         // (5)
                 cs.onCompleted();
             }
         }
 
         @Override
         public void onError(Throwable e) {
             if (once.compareAndSet(false, true)) {
                 csub.unsubscribe();
                 cs.onError(e);
             }
         }
 
         void onTimeout() {
              if (once.compareAndSet(false, true)) {         // (6)
                  csub.clear();
                   
                  other.subscribe(new CompletableSubscriber() {
                      @Override
                      public void onSubscribe(Subscription s) {
                          csub.add(s);
                      }
 
                      @Override
                      public void onCompleted() {
                          cs.onCompleted();
                      }
 
                      @Override
                      public void onError(Throwable e) {
                          cs.onError(e);
                      }
                  });
              }
         }
     });
}
~~~

这个操作符就更复杂一些了：

1. 我们需要追踪前后两个 Completable 的 Subscription，以及用于触发超时的 worker；
2. 由于第一个 Completable 的终止事件和超时事件存在竞争，我们需要用一个原子变量来控制只有其中之一胜出（就像 `amb()` 那样）；
3. 当第一个上游的 Subscription 到来时，我们创建一个容器，并把 Subscription 加入到容器中，然后创建一个 worker，再把容器传递给下游；
4. 最后，我们调度一次 onTimeout 的执行。这一设定让我们避免了 onTimeout 和 onSubscribe 之间的竞争，例如，如果 schedule 发生在 timeout() 被调用时，那超时就有可能在 Subscription 到来之前发生，那我们就需要做一些额外的事情以确保不出问题（这里不详细展开）。大多数时候，在 Reactive-Stream 兼容的 RxJava 2.x 中使用这种风格的 onSubscribe 实现，能够减少很多让我们头疼的事情；
5. 如果第一个 Completable 的终止事件先来，我们就原子地、有条件地设置标记变量（这能阻止 onTimeout 执行其内部逻辑）。由于容器也管理着 worker，所以我们在取消了容器之后，就把终止事件发往下游；
6. 如果 onTimeout 先发生，并且成功设置了标记变量，那我们就把容器清空掉，然后用一个全新的 CompletableSubscriber 订阅到第二个 Completable 中。这里我们之所以先清空容器，是为了容纳第二个 Completable 的 Subscription，而如果不清空，一个已经被取消的容器就不能使用了。清空容器的第二个好处是，它也会取消第一个 Completable，以及进行调度的 worker；

## Hot Completable?

Hot Completable 仅仅是这里的最后一点思考，我并不确定它（或者一个 published Completable）在此之外是否有实际使用场景，但我们在这里还是看看如何实现它。由于 Completable 并不会发出数据，所以只需要实现一种 CompletableSubject 即可，它不会重放任何数据，只重放终止事件。

首先看看 CompletableSubject 的结构：

~~~ java
public final class CompletableSubject 
extends Completable implements CompletableSubscriber {
     
    public static CompletableSubject create() {
        State state = new State();
        return new CompletableSubject(state);
    }
 
    static final class State 
    implements CompletableOnSubscribe, CompletableSubscriber {
 
        // TODO state fields
 
        boolean add(CompletableSubscriber t) {
            // TODO implement
        }
 
        void remove(CompletableSubscriber t) {
            // TODO implement
        }
 
        @Override
        public void call(CompletableSubscriber t) {
            // TODO implement
        }
         
        @Override
        public void onSubscribe(Subscription d) {
            // TODO implement
        }
         
        @Override
        public void onCompleted() {
            // TODO implement
             
        }
         
        @Override
        public void onError(Throwable e) {
            // TODO implement
             
        }
    }
 
    static final class CompletableSubscription 
    extends AtomicBoolean implements Subscription {
        /** */
        private static final long serialVersionUID = 
            -3940816402954220866L;
         
        final CompletableSubscriber actual;
        final State state;
         
        public CompletableSubscription(
                CompletableSubscriber actual, State state) {
            this.actual = actual;
            this.state = state;
        }
         
        @Override
        public boolean isUnsubscribed() {
            return get();
        }
         
        @Override
        public void unsubscribe() {
            if (compareAndSet(false, true)) {
                state.remove(this);
            }
        }
    }
     
    final State state;
     
    private CompletableSubject(State state) {
        super(state);
        this.state = state;
    }
     
    @Override
    public void onSubscribe(Subscription d) {
        state.onSubscribe(d);
    }
     
    @Override
    public void onCompleted() {
        state.onCompleted();
    }
     
    @Override
    public void onError(Throwable e) {
        state.onError(e);
    }
}
~~~

它的结构和[以前的 Subject](/AdvancedRxJava/2016/10/04/subjects-part-2/) 别无二致。我们必须通过工厂方法创建 CompletableSubject，并且需要 CompletableSubscriber 的方法把事件委托到共享的 State 对象上。CompletableSubscription 用于追踪每个 CompletableSubscriber，并根据它们的标识管理取消订阅。

State 类需要记录一个终止标记，以及一个可能发生的 Throwable，加上当前的下游 CompletableSubscriber 数组：

~~~ java
Throwable error;
 
volatile CompletableSubscription[] subscribers = EMPTY;
 
static final CompletableSubscription[] EMPTY = 
        new CompletableSubscription[0];
static final CompletableSubscription[] TERMINATED =
        new CompletableSubscription[0];
 
boolean add(CompletableSubscription t) {
    if (subscribers == TERMINATED) {
        return false;
    }
    synchronized (this) {
        CompletableSubscription[] a = subscribers;
        if (a == TERMINATED) {
            return false;
        }
         
        CompletableSubscription[] b = 
            new CompletableSubscription[a.length + 1];
        System.arraycopy(a, 0, b, 0, a.length);
        b[a.length] = t;
        subscribers = b;
        return true;
    }
}
 
void remove(CompletableSubscription t) {
    CompletableSubscription[] a = subscribers;
    if (a == EMPTY || a == TERMINATED) {
        return;
    }
     
    synchronized (this) {
        a = subscribers;
        if (a == EMPTY || a == TERMINATED) {
            return;
        }
         
        int j = -1;
        for (int i = 0; i < a.length; i++) {
            if (a[i] == t) {
                j = i;
                break;
            }
        }
         
        if (j < 0) {
            return;
        }
        if (a.length == 1) {
            subscribers = EMPTY;
            return;
        }
        CompletableSubscription[] b = 
            new CompletableSubscription[a.length - 1];
        System.arraycopy(a, 0, b, 0, j);
        System.arraycopy(a, j + 1, b, j, a.length - j - 1);
        subscribers = b;
    }
}
~~~

add 和 remove 的实现与其他 Subject 的实现一模一样，主要用于维护当前的下游 CompletableSubscriber 数组。

接下来让我们处理新来的 Subscriber：

~~~ java
@Override
public void call(CompletableSubscriber t) {
    CompletableSubscription cs = 
        new CompletableSubscription(t, this);
    t.onSubscribe(cs);
     
    if (add(cs)) {
        if (cs.isUnsubscribed()) {
            remove(cs);
        }
    } else {
        Throwable e = error;
        if (e != null) {
            t.onError(e);
        } else {
            t.onCompleted();
        }
    }
}
~~~

我们为每个 CompletableSubscriber 创建一个 CompletableSubscription，用来保存 subscriber 以及当前的状态：如果 child 调用了 unsubscribe，那它就可以把自己从 state 的 subscriber 数组中移除了。注意，这里存在 add 和取消订阅的竞争，这会导致 CompletableSubscriber 被意外保留。所以我们在 add 之后检查是否取消订阅，如果已经取消，我们就再调用 remove。如果 add 返回了 false，这说明 CompletableSubject 已经处于终止状态，读取 error 成员我们可以判断是否发生了异常，并确定向下游发送何种事件。

处理 onXXX 也不复杂：

~~~ java
@Override
public void onSubscribe(Subscription d) {
    if (subscribers == TERMINATED) {
        d.unsubscribe();
    }
}
 
@Override
public void onCompleted() {
    CompletableSubscription[] a;
    synchronized (this) {
        a = subscribers;
        subscribers = TERMINATED;
    }
     
    for (CompletableSubscription cs : a) {
        cs.actual.onCompleted();
    }
}
 
@Override
public void onError(Throwable e) {
    CompletableSubscription[] a;
    synchronized (this) {
        a = subscribers;
        error = e;
        subscribers = TERMINATED;
    }
     
    for (CompletableSubscription cs : a) {
        cs.actual.onError(e);
    }
}
~~~

onSubscribe 的行为倒是值得商榷，这里我的做法是，如果 CompletableSubject 已经处于终止状态，就将其取消订阅。即便不这样，我们也不能做太多事，因为 CompletableSubject 可能订阅到很多 Completable，任何一个都可能让其进入到终止状态。我们也可以忽略传入的 Subscription，或者把它们都保存到一个容器中。

onError 和 onCompleted 的处理基本一致，我们原子地把 subscriber 数组设置为终止标记，然后向此前的每个 subscriber 发出相应的终止事件。注意，在 onError 中，我们先设置 error 成员，再设置 subscribers 成员，这让我们可以在前面的 call() 函数中得到恰当的可见性。

## 总结

在本文中，我详细展示了如何实现 Completable 操作符，既包括数据源操作符，也包括转换型操作符，甚至还实现了一个 Subject。

Completable 更像是一个类型工具，它不会发出任何事件，只是表示有“副作用”的代码执行是否成功。简化了的 API 用起来可能比 Observable 更加方便。

实现 Completable 操作符比实现支持 backpressure 的 Observable 操作符更简单，但我们还是需要考虑取消订阅链条、避免 onXXX 的竞争，以及利用好 `AtomicXXX` 类型以更高效地实现状态管理。

既然我们已经有了更多编写 Subject 的经验，下一个系列让我们看看 ConnectableObservable。
