---
layout: post
title: "HFL: Hybrid Fuzzing on the Linux Kernel"
date: 2020-05-09 14:56:44 +0800
comments: true
categories: 
---

> 作者：Kyungtae Kim^1^, Dae R. Jeong^2^, Chung Hwan Kim^3^, Yeongjin Jang^4^, Insik Shin^2^, and Byoungyoung Lee^5^ ^1^
>
> 单位：^1^Purdue University, ^2^KAIST, ^3^NEC Laboratories America, ^4^Oregon State University, and ^5^Seoul National University
>
> 出处：NDSS '20
>
> 原文：<https://www.ndss-symposium.org/wp-content/uploads/2020/02/24018-paper.pdf>

## Abstract

将符号执行（symbolic execution）与模糊测试（fuzzing）相结合以达到优势互补的混合模糊测试（hybrid fuzzing）是进行漏洞挖掘的一种有效手段。然而，直接将混合模糊测试应用在内核测试中的效果并不好，这是由于内核自身具有的下列特性导致的：1）存在由系统调用参数决定的间接跳转；2）通过系统调用控制和匹配内部系统状态；3）推断系统调用的嵌套参数类型。尽管这些特性对于模糊测试和符号执行来说也是十分重要的，但是现有的内核测试手段最多只是通过静态分析的方式不精确的处理了的这些特性中的一小部分。

因此，本文提出了内核混合模糊测试系统HFL，其通过下列技术解决了内核特有的模糊测试难点：1）将间接跳转转化为直接跳转；2）推断系统调用顺序以构造一致的系统状态；3）识别系统调用的嵌套参数类型。结果表明，HFL在最近的Linux内核中发现了24个之前未知的漏洞。相比于Moonshine与Syzkaller，HFL的代码覆盖率分别提升了15%与26%。相比于kAFL/S2E/TriforceAFL，HFL的代码覆盖率甚至提升了四倍。此外，针对13个已知的漏洞，HFL的漏洞挖掘效率与Syzkaller相比提升了三倍多。

<!-- more -->

## 1 Introduction & Background

模糊测试是一项通过生成随机输入测试目标程序的技术。然而，由于其内在的局限性，模糊测试无法探索位于困难分支条件之后的程序路径，例如，$if (i == 0xdeadbeef)$，因为这需要在巨大的搜索空间中猜到单一的值。

符号执行是一项通过生成符号输入驱动目标程序运行至特定程序路径的技术。符号执行的一个主要局限在于其面临的状态爆炸问题（state explosion problem）。每当遇到一个条件分支时，符号执行便需要复制程序状态以同时探索这个分支两侧的路径。

结合了模糊测试与符号执行的混合模糊测试便可以补足上述两项技术各自的局限性。一方面，模糊测试经常受到困难分支条件的限制，然而，符号执行却能够提供可以探索这些分支的输入。另一方面，符号执行可能遭遇状态爆炸的问题，然而，模糊测试却能够利用经过测试的输入来引导符号执行只探索特定的程序路径。

内核自身具有的特性导致了直接将混合模糊测试应用在内核测试中是十分困难的。作者总结了下列三个内核特有的模糊测试难点：

+ Linux内核大量利用间接跳转以支持多态，导致传统模糊测试的效果并不好；
+ 推断系统调用的正确顺序与依赖关系以构造满足漏洞触发要求的系统状态是十分困难的；
+ 系统调用参数中的嵌套结构使得接口模板化困难。

为了解决这些挑战，本文提出了HFL，其基于内核模糊测试系统Syzkaller与符号执行引擎S2E实现，利用混合模糊测试挖掘Linux内核中的漏洞，并通过下列三项技术解决了前述内核特有的混合模糊测试挑战：

+ HFL通过在编译期间转化原始内核，将间接跳转转化为直接跳转；
+ HFL通过预先执行静态指向分析（static points-to analysis），选择性的符号化与系统状态相关的数据变量，以推断正确的系统调用顺序，并重建系统状态；
+ HFL根据内核处理系统调用参数的方式，在运行期间恢复嵌套参数类型。

## 2 Motivation

作者总结了下列三个将混合模糊测试应用在内核上的挑战：

+ 由输入决定的间接跳转；
+ 内部系统状态要求；
+ 嵌套参数类型推断。

这些挑战并不只限于混合模糊测试，他们同样会影响单独执行模糊测试或者符号执行的技术的效果。尽管如此，他们中的大部分还是没有在现有的模糊测试工作中得到很好的解决。

![20200425224531](/images/2020-05-09/20200425224531.png)

### A. Indirect Control Transfer Determined by Input

为了支持大量不同的设备和特性，即利用单一的接口支持多态，Linux内核中的大部分组件都被分离成了抽象接口与实现两层，接口层通常被用来访问特定的实现。具体来说，Linux内核通常会构造一张函数指针表来作为抽象接口，其包含指向各个具体实现的函数指针。

![20200425224707](/images/2020-05-09/20200425224707.png)

### B. Internal System States

内核通过维护内部系统状态以管理计算资源。这些内部系统状态中的大部分都是主要通过系统调用来控制的。由于系统调用与系统状态的高度相关，如果在没有正确构造系统状态的前提下执行系统调用，那么这个系统调用将会被内核直接拒绝。

为了提升代码覆盖率，作者考虑了两类基本情况：1）系统调用的正确执行顺序需要得到维护；2）系统调用参数之间的依赖关系同样需要得到保持。

![20200425225006](/images/2020-05-09/20200425225006.png)

### C. Nested Syscall Arguments

系统调用参数经常被构造成嵌套结构，即结构中一个数据成员指向另一个结构，另一个数据成员标识下一个嵌套结构的大小。在很多情况下，这样的嵌套结构的精确定义只有在运行期间才能得到。

![20200425225046](/images/2020-05-09/20200425225046.png)

## 3 Design & Implementation

![20200425225109](/images/2020-05-09/20200425225109.png)

### A. Generic Hybrid Fuzzing Design

总的来说，HFL的设计沿用了现有的针对用户态程序的混合模糊测试实现，将传统的模糊测试与符号执行相结合以达到优势互补。

HFL的模糊测试特性在大体上可以被总结成以下三点：

+ 针对内核系统调用的模糊测试；
+ 代码覆盖率导向的模糊测试；
+ 符号执行。

与Syzkaller的实现类似，HFL的模糊测试方案集中在生成并变异由一系列系统调用组成的用户态程序。类似的，HFL同样也沿用了大部分其它模糊测试系统采用的代码覆盖率导向的模糊测试方案，其模糊测试输入变异策略是以提升代码覆盖率为主要目标的。

在模糊测试的过程中，HFL的模糊测试引擎能够识别内核中的困难分支条件，即在多次用户态程序的执行过程中取值均为真或者假的分支条件。为了检测这样的困难分支条件，HFL在模糊测试的过程中维护了一张频率表记录各个分支取值为真或者假的次数。

值得注意的是，在模糊测试的过程中，负责程序路径探索的是HFL的模糊测试引擎，HFL的符号执行引擎只需要沿着一条单一的程序路径按需执行约束求解，避免了路径爆炸的问题。

### B. Converting Control-Flow from Indirect to Direct

![20200425225203](/images/2020-05-09/20200425225203.png)

HFL设计了一个离线转化工具，其基于内核的源代码，将间接跳转转化为直接跳转，具体可分为以下两步：1）转化工具验证函数指针表的索引变量来源于系统调用参数；2）根据函数指针表和可能的索引值，HFL执行分支转化，针对各个索引值插入跳转至相应函数指针的条件分支。

### C. Consistent System State via Syscall Sequence Inference

![20200425232724](/images/2020-05-09/20200425232724.png)

为了推断系统调用的正确顺序与依赖关系，HFL首先针对内核执行静态分析以搜集候选的依赖关系。基于这些候选的依赖关系，HFL随即针对内核进行符号执行以验证并筛选出正确的依赖关系。接着，HFL通过追踪经过符号化的系统调用参数之间依赖关系的传播，以检测系统调用参数取值之间的依赖关系。最终，HFL将会把推断出的系统调用的正确顺序反馈给模糊测试引擎，以立即应用在未来的用户态程序变异中。

### D. Nested Syscall Argument Retrieval

![20200425232748](/images/2020-05-09/20200425232748.png)

HFL根据内核处理系统调用参数的方式，即调用数据转移函数$copy\_from\_user$与$copy\_to\_user$，并结合符号执行以理解并恢复系统调用的嵌套参数类型。其中，嵌套输入结构指向的内存区域的地址与大小是恢复嵌套系统调用参数的关键。

### E. Implementation

HFL是基于现有的模糊测试系统Syzkaller与符号执行引擎S2E实现的。作者在gcc的中间语言GIMPLE上实现了将间接跳转转化为直接跳转的功能，并利用静态分析框架SVF实现了针对Linux内核源代码的跨过程静态分析。

## 4 Evaluation

本章评估了HFL的效果与性能，旨在回答以下四个研究问题：

+ **Q1：**HFL挖掘内核漏洞的效果如何？
+ **Q2：**相比于现有的内核测试手段，HFL带来的总体代码覆盖率的提升有多少？
+ **Q3：**相比于其它现有的模糊测试系统，HFL挖掘内核漏洞的性能如何？
+ **Q4：**HFL采用的各项技术对于总体性能的贡献各有多少？

![20200425203935](/images/2020-05-09/20200425203935.png)

基于内核中各个子系统的功能特性，与不同系统调用接口的相关性，以及存在漏洞的可能性，作者将内核中的子系统分为以下三类：1）网络；2）文件系统；3）设备驱动。

![20200425232909](/images/2020-05-09/20200425232909.png)

HFL总共发现了51个漏洞，其中有24个漏洞是之前未知的。作者提交了所有新发现的漏洞，其中有17个漏洞已经得到了确认。

![20200425232837](/images/2020-05-09/20200425232837.png)

HFL和Syzkaller都发现了相同的13个之前已知的漏洞。然而，相比于Syzkaller花费了超过50个小时才发现所有这些漏洞，HFL只花费了约15个小时，这说明HFL拥有更高的漏洞挖掘效率。

![20200425233206](/images/2020-05-09/20200425233206.png)

![20200425233101](/images/2020-05-09/20200425233101.png)

为了说明HFL的效果，作者将其与其它五个现有的内核测试系统进行了比较，即Syzkaller，S2E，Moonshine，kAFL，与TriforceAFL。

值得注意的是，HFL针对设备驱动子系统的代码覆盖率提升最大，这主要是因为这个子系统的实现中使用了大量的$ioctl$函数，这个函数的第二个参数$cmd$通常难以被预测，同时，第三个参数$arg$通常也是未知的各种大小的嵌套参数类型。

![20200425233304](/images/2020-05-09/20200425233304.png)

**F-N**代表作为基准的模糊测试系统，**F-H**代表具有混合模糊测试特性的模糊测试系统，**F-I**代表具有间接跳转处理特性的模糊测试系统，**F-C**代表结合了系统调用顺序推断与参数接口恢复特性的模糊测试系统。

混合模糊测试特性对于总体代码覆盖率的贡献最大，尽管其它的特性也同样不可或缺。作者强调，对于最大化提升总体代码覆盖率，HFL的每个特性都十分关键。
