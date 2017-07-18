---
layout: post
title: Python/Java 才是王道，PHP 还得靠边站
date: 2017-07-18 15:01:02 +0800
comments: true
categories: 
---

3 月份 Erik Bernhardsson 发表了一篇文章 [The eigenvector of "Why we moved from language X to language Y”](https://erikbern.com/2017/03/15/the-eigenvector-of-why-we-moved-from-language-x-to-language-y.html) ，作者根据 Google 里面搜索 `move from <language 1> to <language 2>` 或 `switch to <language 2> from <language 1>` 等的结果数量，做了一个编程语言之间的转移矩阵（Stochastic matrix），采用马可夫链的平稳分布，得到了一些关于未来什么语言最流行的结论，文中也多次提到，那就是 go 语言，很多人搜索了从 xx 到 go ，go 语言才是未来的语言。此外，作者认为 Perl 已死，Java 正当年，Rust 也很有希望。

当然，在 Google 搜一下从 xx 到 yy ，并不代表真的开发者就要移情别恋，只有真的写了 yy 的代码，新的恋情才算开始。

所以，上周 Waren Long 基于 GitHub 数据的统计结果，发表了一篇文章 [Analyzing GitHub, how developers change programming languages over time](https://blog.sourced.tech/post/language_migrations/) ，该文章统计了包含GitHub 中 450 万用户，以及10 TB 的源代码，分析了 393 种语言的变化趋势，最后得出的结论就是：Python 和 Java 才是王道。

这是 Waren Long 总结出来的流行语言的 Top 10 ：

![](/images/2017/07/language_top_10.png)

从上图可以看到 Python (16.1 %) 才是最流行的编程语言，紧随其后的是 Java (15.3 %) 。而且前五名中，Python 代码量也比其他语言所占的比例要低（只有 11.3 %，我们可以理解为投入产出比、效益率，投入越少，产出越大，当然语言就比较流行），PHP 代码量最大，达到了 24.4 %，但是其流行程度只有 Python 的一半而已。

在这个排名中，go 排名第 9 ，但是相对于其 0.9 % 的代码量占比，这个位置还是很牛的了，效益率也是最高的。考虑到这个数据的时效性，相信 go 语言肯定还会进一步扩大受众。


下图是各语言之间的转换矩阵，左面是 From ，右面是 To ：

![](/images/2017/07/sum_matrix_22lang_eig.svg)

在排名前 5 的语言中（Java、 C、 C++、 PHP、 Ruby，除去 Python ），有平均近 22% 的概率，会转移到 Python，不过大叔不理解的是为啥 Ruby 还会转移到 Python ？

而作为统计学领域中的 Fortran (36 %)、 Matlab (33 %) 和 R (40 %) ，转移到 Python 的概率都是非常高的，可见，在数学计算、机器学习等领域，Python 的地位似乎越来越稳固了。

从上面的转移矩阵，我们还可以发现一些有意思的规律，比如：

* Pascal 程序员最喜欢学习 Java
* Visual Basic 程序员学习 C# 顺理成章，C# 程序员爱转 Python
* Scala/Clojure 程序员最喜欢 Java，但是反过来不是，Java 程序员爱转 Python
* Object C 和 Swift 互转说明这个统计结果还是挺靠谱的
* 转 go 最多的竟然不是 PHP 这样的脚本语言，而是 Rust，难道是 Rust 的学习曲线太陡了？

下面的图是 16 年来各语言的平稳分布图：

![](/images/2017/07/eigenvect_stack_22lang.png)

不过大叔不知道这16年的数据是怎么来的，如果我没记错的话，那时候应该还没有 GitHub 。

前两名 Python 和 Java 的趋势基本一样，都是稳中有升的形状。这两个语言已经取代了 C 语言的霸主地位，如果把这 3 种语言合起来看，则它们的总量几乎没什么变化。

Perl 基本死了，PHP 的扩张趋势也没那么明显，Ruby 粗了几年又细了，应该是和 Python 份额的扩大和 go 等的日渐流行有关。

如果让我总结一下，那么 Java、Python 和 go 值得关注、持续学习。

<hr>

这就是 Waren Long 原文里的几个重要结论，如果你对原始数据和算法感兴趣，可以直接 [阅读原文](https://blog.sourced.tech/post/language_migrations/)  。

*注意： 由于 JavaScript 的特殊性（前后通吃、受众广等，不适用于统计模型），Waren Long 的文章中并没有将 JavaScript 作为评价对象。*

