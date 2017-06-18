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

## 最初的 Stream API 基准测试

也许理解这一基准测试最简单的办法，就是看看最初非并行的 Stream 版本。由于串行或并行只需要对 Stream 使用 `sequential()` 或 `parallel()` 操作符即可，所以这两种情况继承自同一个基类，其中包含了主要的代码：[ShakespearePlaysScrabbleWithStreamsBeta.java](https://github.com/akarnokd/akarnokd-misc/blob/master/src/jmh/java/hu/akarnokd/comparison/scrabble/ShakespearePlaysScrabbleWithStreamsBeta.java)。

我增加了 Beta 后缀，表明在这里它是一个替代版本，用于区分另一个计算步骤上稍有差异的版本。稍后我讲解 RxJava 基准测试时我将解释为什么要这样。

这个基准测试以一种不太符合常规，甚至过度函数式的方式实现，这让我理解它的功能花了不少脑细胞。尽管它也不是特别复杂。

基准测试的输入藏在基类的 `shakespeareWords`（`HashSet<String>`）、`scrabbleWords`（`HashSet<String>`）、`letterScore`（`int[]`）和 `scrabbleAvailableLetters`（`int[]`）成员中。`shakespeareWords` 包含了所有的单词（取自莎士比亚剧本，全小写）。`scrabbleWords` 包含了所有 Scrabble 允许的单词，也是小写。`letterscore` 是每个字母的分数，`scrabbleAvailableLetters`（我认为）是用于同一字母出现多次时限制得分的。

这个基准测试由于每一步之间存在依赖，所以实际是反向编写的，首先是获取字母分数的函数。给定一个取值范围为 96~121 的英文字母，该函数将其映射为 0~25，再从数组中取出分数：

~~~ java
IntUnaryOperator scoreOfALetter = letter -> letterScores[letter - 'a'];
~~~

第二个函数是给定一个单词的字母直方图（以 `Map.Entry` 形式，key 是字母，value 是字母出现次数），计算出这个单词中该字母的总分数。

~~~ java
ToIntFunction<Entry<Integer, Long>> letterScore =
    entry ->
        letterScores[entry.getKey() - 'a'] *
        Integer.min(
            entry.getValue().intValue(),
            scrabbleAvailableLetters[entry.getKey() - 'a']
        );
~~~

为了使用这个函数，我们就需要一个计算字母直方图的函数：

~~~ java
Function<String, Map<Integer, Long>> histoOfLetters =
    word -> word.chars()
                .boxed()
                .collect(
                    Collectors.groupingBy(
                        Function.identity(),
                        Collectors.counting()
                    )
                );
~~~

这就是每个数据流处理库发挥作用的地方了。给定一个 Java `String`，将其分割为单独的字母，然后统计每个字母出现的次数（_例子省略_）。在 Stream 版本中，`String` 提供的 `IntStream` 被转换为了 `Stream<Integer>`，然后组合为 `Map`。注意这个函数的返回值是 `Map` 而非 `Stream<Map>`。

下一个函数是取得有范围限制的字母得分：

~~~ java
ToLongFunction<Entry<Integer, Long>> blank =
    entry ->
        Long.max(
            0L,
            entry.getValue() -
                scrabbleAvailableLetters[entry.getKey() - 'a']
        );
~~~

给定一个直方图的 entry，我们从中减去 `scrabbleAvailableLetters` 中的次数，当然不得小于 0。

下一个函数结合 `histoOfLetters` 和 `blank`：

~~~ java
Function<String, Long> nBlanks =
    word -> histoOfLetters.apply(word)
                          .entrySet().stream()
                          .mapToLong(blank)
                          .sum();
~~~


