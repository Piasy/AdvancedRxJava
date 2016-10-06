---
layout: post
title: 深入理解 Operator：Amb 和 AmbWith
tags:
    - Operator
---

原文 [Operator internals: introduction](http://akarnokd.blogspot.com/2015/10/operator-internals-amb-ambwith.html){:target="_blank"}

## 介绍

`amb` 是 `ambiguous` 的缩写，它的输入是一个 `Observable` 集合，输出的是转发第一个发出事件的 Observable 的所有后续事件，并且取消订阅其他所有的 Observable，也会忽略它们的任何事件。amb 支持 backpressure。


