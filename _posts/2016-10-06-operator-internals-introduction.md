---
layout: post
title: 深入理解 Operator：开篇
tags:
    - Operator
---

原文 [Operator internals: introduction](http://akarnokd.blogspot.com/2015/10/operator-internals-introduction.html){:target="_blank"}

开发操作符通常来说都不是一个傻瓜式的活。在这个 Advanced RxJava 系列博文中，我尽可能多地涵盖了我们用到的一些基础内容，以及在开发操作符的过程中得到的经验。

但是 RxJava 有将近 150 个不同的操作符，而它们中的大多数都有些自定义的逻辑，或者说超出常规的逻辑。这些内容都是很难甚至不可能用通用的方式进行讲解的。

所以后面我会开启众多小的系列，来逐个深入分析讲解操作符，当然有些一目了然的操作符就不会讲了。不管它们实现的时候是以 `Operator` 的形式还是 `OnSubscribe` 的形式，我都称之为操作符，不同的形式是为了代码上更便捷。我将会按照字母序遍历所有的操作符，并且在同一篇文章中，顺带把所有名字不同功能一样的操作符一起进行讲解。

我会同时讲解 RxJava 1.x 和 2.x 中的操作符，这样做有两个好处，一是作为对代码实现的一次 review，另外也可以看到我们是怎么让操作符遵循 Reactive-Streams 规范的。
