---
layout: post
title: "Pangolin: Incremental Hybrid Fuzzing with Polyhedral Path Abstraction"
date: 2020-10-30 15:36:24 +0800
comments: true
categories: 
---

> 作者：Heqing Huang^1^, Peisen Yao^1^, Rongxin Wu^2^, Qingkai Shi^1^, and Charles Zhang^1^
>
> 单位：^1^The Hong Kong University of Science and Technology and ^2^Xiamen University
>
> 出处：S&P '20
>
> 原文：<https://qingkaishi.github.io/public_pdfs/SP2020.pdf>

## Abstract

混合模糊测试（hybrid fuzzing）是一种结合了覆盖率导向的模糊测试（coverage-guided fuzzing）与动态符号执行（concolic execution）两者优势的技术。尽管该技术已经得到了广泛的研究，但作者认为现有实现依旧存在效率低下的问题。这是由于这些实现都是非增量的（non-incremental），它们只缓存并重用极少部分的路径约束求解结果。

为了实现增量的混合模糊测试，作者提出了多边形路径抽象（polyhedral path abstraction），该技术通过保存动态符号执行探索路径约束的求解结果来提升后续模糊测试种子变异与路径约束求解的效率。作者基于该技术实现了Pangolin，并在LAVA-M数据集和九个真实程序上对其进行了评估。与现有模糊测试实现相比，Pangolin的路径覆盖率有10%至30%的提升，其在LAVA-M数据集中多找到了400个程序错误。此外，Pangolin在真实程序中发现了41个未被发现的程序错误，其中包括8个新的CVE漏洞。

<!-- more -->

## 1 Introduction

尽管混合模糊测试可以达到极高的路径覆盖率，但其效率问题却依旧突出。这是由于现有实现在探索嵌套的路径约束时都是非增量的，只有极少部分的路径约束求解结果会被保存并重用在后续的模糊测试种子变异与动态符号执行上。然而，当前阶段的路径约束求解结果是可以为后续阶段的探索提供指导的。

在当前的混合模糊测试实现中，动态符号执行探索路径约束的求解结果是不会在各个阶段之间保存并重用的，这就造成了非常严重的计算冗余。

![](/images/2020-10-30/uploads/upload_99a8cfd415af3f1d5d9c8b4ba1942e7f.png)

增量的混合模糊测试通过保存并重用当前阶段的路径约束求解结果来引导后续阶段的模糊测试种子变异与路径约束求解。作者在动态符号执行的过程中通过多边形路径抽象来线性化路径约束为多边形，从而推断输入变量与它们线性表达式的取值范围：

+ 多边形路径抽象将模糊测试种子的变异问题转化为在多边形上的取样问题，作者采用Dikin漫步算法来在遵守目标路径约束的前提下高效地生成大量新的模糊测试种子。
+ 作者采用多边形路径抽象来加速动态符号执行的路径约束求解。首先，由于多边形路径抽象是目标路径约束可靠的简化形式，作者可以通过重用前驱路径的多边形路径抽象来可靠地推断当前路径的可达性，从而极大地降低路径约束求解的复杂度。其次，作者还可以通过重用前驱路径的多边形路径抽象来缩小当前路径约束的解空间。

## 2 Background

覆盖率导向的模糊测试实现一般包含三个步骤：

1. 确定未被覆盖的路径。
2. 生成满足这些路径约束的模糊测试种子。
3. 变异这些模糊测试种子来生成大量新的模糊测试输入以进一步提升路径覆盖率。

大部分的模糊测试实现采用一些简单的启发式规则来随机地变异模糊测试种子。为了缩小模糊测试输入巨大的搜索空间，一些实现采用污点分析来确定模糊测试输入中与目标路径约束相关的部分。另外一些实现采用了混合模糊测试，该技术通过路径约束求解来生成可以满足复杂路径约束的模糊测试种子。

![](/images/2020-10-30/uploads/upload_873426594b7bea26f3f8c082d3cdbe55.png)

图中的黑点表示满足目标路径约束的所有模糊测试种子的取值，十字表示不满足的模糊测试种子的取值。路径抽象则是图中由表示多个线性不等式的线条围成的区域。路径抽象是非线性路径约束的近似，图中所有的路径抽象都是可靠的，因为其覆盖了所有的黑点。

## 3 Overview & Methodology

![](/images/2020-10-30/uploads/upload_829d3c13648e08401e0b386644c4a9dc.png)

Pangolin本质上是一种混合模糊测试实现，其具有传统混合模糊测试的典型组件，例如模糊测试种子优先级排序与模糊测试执行追踪。与传统混合模糊测试不同，Pangolin的包含三个核心组件：1）路径抽象构建；2）有约束的模糊测试种子变异；3）有引导的路径约束求解。

### 3.1 Path Abstraction Inference

作者首先识别出在模糊测试的过程中未被覆盖的路径，然后调用动态符号执行引擎来构建这些路径的多边形路径抽象。这些抽象可靠地描述了满足目标路径约束的模糊测试输入的解空间，其可以被用来引导后续阶段的模糊测试种子变异与路径约束求解。

多边形路径抽象是目标路径约束简化的线性近似。为了支持增量的混合模糊测试，目标路径约束的多边形路径抽象应当满足两点要求：

+ 这些路径抽象应当是可靠的，可以覆盖满足目标路径约束的所有模糊测试种子的取值。
+ 这些路径抽象可以被高效构建，覆盖的不满足目标路径约束的模糊测试种子的取值应当尽可能的少。

![](/images/2020-10-30/uploads/upload_8e267eed80f766ab4d54b069670f1bc6.png)

作者基于BWAI算法来将目标路径约束线性化为多边形的形式，该算法将目标路径约束的多边形路径抽象构建视为典型的基于SMT的优化问题，以可靠地计算线形表达式在满足目标路径约束前提下的最小上界。

BWAI算法采用预先定义的模板来枚举任意两个输入变量之间所有可能的线形关系以作为目标线形表达式，例如$x_1$、$x_2$、$x_1 + x_2$与$x_1 - x_2$。为了降低多边形路径抽象构建的开销，作者并不生成所有可能的线形不等式，他们只计算单个输入变量和存在于目标路径约束中的线形表达式的取值范围。

![](/images/2020-10-30/uploads/upload_c022d896b8e9ce1195e215263fc846b6.png)

举例来说，针对目标路径约束$x \leq 2 \land y \leq 5 \land x^2 - 5x +4 \leq y$，作者构建的多边形路径抽象为$0 \leq x \leq 2$、$0 \leq y \leq 5$与$4 \leq 5x + y \leq 15$。该路径抽象是目标路径约束精确且简化的近似。

![](/images/2020-10-30/uploads/upload_adaf95b85781aaf81a73668e9bf100ca.png)

### 3.2 Constrained Mutation

多边形路径抽象提供了满足目标路径约束的输入变量的取值范围，因此作者可以通过在这些取值范围上取样来快速地生成大量新的模糊测试输入，从而进一步探索具有相同前驱路径的新路径。作者采用了Dikin漫步算法来生成新的模糊测试输入，该算法可以保证生成的模糊测试输入均匀地覆盖目标路径约束的解空间，从而保证探索程序状态的多样性。

模糊测试种子的变异算法应当具有两方面的属性：

+ 该算法不能是完全随机的，生成的模糊测试输入应当满足目标路径约束，从而可以高效地为当前路径生成大量新的模糊测试输入。
+ 该算法生成的模糊测试输入应当均匀地散布在目标路径约束的解空间中，这样才更有可能触发不同的程序行为。

![](/images/2020-10-30/uploads/upload_055f0a0187223f5fb72b66edff3c5f69.png)

Pangolin采用Dikin漫步算法来保证高效且均匀的模糊测试种子变异。该算法的时间复杂度是$O(nm)$，其中$m$和$n$分别是多边形路径抽象中不等式和变量的个数。该算法多项式的时间复杂度要远低于路径约束求解NP完全的时间复杂度。

为了确定模糊测试种子变异的次数，作者考虑了两方面的因素：1）为难以被覆盖的路径生成更多的模糊测试输入；2）为具有更多分支的路径生成更多的模糊测试输入。

作者只有在面对难以被覆盖的路径时才会采用有约束的模糊测试种子变异。在其它情况下，作者仍然采用随机的模糊测试种子变异。

### 3.3 Guided Constraint Solving

在完成模糊测试输入的取样之后，Pangolin依旧离线保存之前构建的多边形路径抽象以用于后续阶段路径约束求解。与重量级的内存快照等方法不同，多边形路径抽象是一种轻量级的缓存技术，这些抽象是目标路径约束解空间可靠的简化近似，其可以极大地加速具有相同前驱路径的路径约束求解。

多边形路径抽象在可以引导模糊测试种子变异之外还可以从两方面加速开销巨大的路径约束求解：1）快速过滤掉不可达的路径；2）缩小可达路径约束的解空间。

在求解目标路径约束之前，作者首先采用现有前驱路径的多边形路径约束来构建出目标路径约束的简化形式。简化路径约束通常比目标路径约束简单且易于求解。如果该路径约束不能被满足，那么目标路径约束也一定不能被满足，当前路径是不可达的。如果该路径约束可以被满足，那么该路径约束可以进一步缩小目标路径约束的解空间。

![](/images/2020-10-30/uploads/upload_0b2f724f71be14d2978e52b01844bfb3.png)

## 4 Evaluation

Pangolin是基于覆盖率导向的模糊测试框架AFL与动态符号执行框架QSYM构建的。作者设计了一系列的实验来评估Pangolin的有效性，并提出了三个研究问题：

+ Pangolin可以比现有的模糊测试实现检测出更多的程序错误吗？
+ Pangolin可以比现有的模糊测试实现达到更高的路径覆盖率吗？
+ Pangolin采用的有约束的模糊测试种子变异与有引导的路径约束求解这两项技术的有效性如何？

作者在LAVA-M数据集和九个真实程序上对Pangolin进行了评估，并与AFL、AFLFast、QSYM、Driller、Angora与T-Fuzz这六个现有的模糊测试实现进行了比较。

![](/images/2020-10-30/uploads/upload_2351cc9964a182aba65ec06bdcfe8da2.png)

![](/images/2020-10-30/uploads/upload_e04682da43375ed0fc73af8370f5b3b9.png)

为了避免随机性带来的影响，作者将每个实验重复了十次，并将平均值作为最终的结果。作者还采用了Mann-Whitney U检验来证明评估结果的统计显著性。

### 4.1 Discovering Bugs

![](/images/2020-10-30/uploads/upload_a862289658923de33b894b83617b9935.png)

作者进一步分析了Pangolin无法检测LAVA-M数据集中所有程序错误的原因。由于Pangolin是基于动态符号执行框架QSYM构建的，其继承了QSYM的一些局限，例如缺少对底层系统调用的模拟与对浮点数路径约束的支持。这些局限导致Pangolin无法检测程序who中的一些程序错误。

![](/images/2020-10-30/uploads/upload_cc4f062b7b9e626ebe934861e04d2324.png)

### 4.2 Coverage Comparison

作者采用Mann-Whitney U检验来计算评估结果的p值，该检验有两个级别的统计显著性，包括0.01与0.05。

Pangolin需要初始的预热时间来构建多边形路径抽象，因此无法在实验一开始就发挥其最佳性能。

![](/images/2020-10-30/uploads/upload_0c2b1edccc58402476ee90c1b63424c2.png)

### 4.3 Key Feature Evaluation

![](/images/2020-10-30/uploads/upload_479869122edf7f13545c79524e95641d.png)

为了评估Pangolin采用的有约束的模糊测试种子变异的有效性，作者将其与基于SMT的取样框架SMTSampler进行了比较。该框架的目标同样是通过路径约束求解来生成多个满足目标路径约束的变量取值。

![](/images/2020-10-30/uploads/upload_fe9d2e36aeed16bd0584fab6230d04a6.png)

Pangolin采用的取样算法，Dikin漫步算法可以保证生成的模糊测试输入的均匀散步，而SMTSampler却不行。该均匀散布的特性可以提升Pangolin的程序错误检测的有效性。

![](/images/2020-10-30/uploads/upload_8a8ee453078e14ee1ac84c7d1d675261.png)

### 4.4 Case Study

![](/images/2020-10-30/uploads/upload_378d96aac8560948d30c437ba0f55870.png)
