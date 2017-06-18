---
layout: post
title: 响应式编程库 Scrabble 基准测试
tags:
    - 对比点评
---

原文 [The Reactive Scrabble benchmarks](http://akarnokd.blogspot.com/2016/12/the-reactive-scrabble-benchmarks.html){:target="_blank"}。

## 介绍

过去一年多的时间里，我以神秘的莎士比亚 Scrabble 之名发布了很多基准测试结果。在本文中，我将解释这个基准测试是什么、从何而来、如何运作、有何目的，以及如何在未测试的库上面运行这一基准测试。

## 历史

这一基准测试由 [Jose Paumard 设计开发](https://github.com/JosePaumard/jdk8-stream-rx-comparison)，测试结果首次于[他的 2015 Devoxx 演讲](https://www.youtube.com/watch?v=fabN6HNZ2qY)上发布。这一基准用于测量某个数据处理的库能以多快的速度从一个单词集合中找到最有价值的单词，这些单词取自莎士比亚的剧本，单词评分则基于 Scrabble 框架定义的规则。彼时参与测评的是 RxJava 1.0.x 版本，令我吃惊的是，相比于 Java 8 Stream API，RxJava 的性能实在差得离谱：

![](https://imgs.piasy.com/2017-06-18-jose_scrabble_results.png)

这一基准测试利用了 JMH，完全同步运行；在没有任何线程弹跳的情况下 RxJava 运行耗时长 10 倍，或者更可能的是，它使用的一系列操作符有 10 倍的开销。此外，Jose 还增加了一个并行序列的版本，在 join 线程之前，主循环并行执行。

更令人失望的是，RxJava 2 开发者预览版的表现也一样糟糕（[二月份在一个相对弱的 CPU 上运行的测试](https://twitter.com/akarnokd/status/696291409209487360)）。

因此，我并没有埋怨这个基准测试或者其作者，而是尝试理解这一基准测试期望的性能点，并据此优化 RxJava 2 的性能，如有可能还要移植到 RxJava 1.x 上去。

_译者注：描述 benchmark 代码的部分冗长无味，我就不翻译了。_

## 最初测试 Stream API 的基准测试

……

## 最初测试 RxJava 的基准测试

……

## 优化的测试 RxJava 基准测试

……

## 测试其他库

过去一年我们评测了各种库，结果如下图：

![](https://imgs.piasy.com/2017-06-18-Scrabble_12_14.jpg)

_译者注：此处省略各个库测试过程中的一些注意事项。_

## 总结

编写一个响应式编程库很难，为这些库编写基准测试也不简单。要理解为何不同库之间快慢差异之巨，需要对同步异步的设计开发都有深厚的功底。

不像一年前我对 Scrabble 基准测试集恼羞成怒，过去一年里我花了很多精力去改进和优化我能影响到的库，多亏了这些不懈的努力，这些库的架构和概念上都有所改进，它们在这个基准测试集上以及通用场景下都表现更佳了。

我必须警告读者，不要把 Scrabble 基准测试集的结果作为这些库排名的终极指标。它们在这个基准测试集上表现如此，并不代表在其他场景或任务下也表现如此。计算量/IO 的开销也许就会掩盖这一丁点的差异。
