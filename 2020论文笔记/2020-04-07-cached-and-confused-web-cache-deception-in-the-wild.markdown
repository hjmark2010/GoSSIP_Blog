---
layout: post
title: "Cached and Confused Web Cache Deception in the Wild"
date: 2020-04-07 11:08:16 +0800
comments: true
categories: 
---

> 会议：USENIX 2020
>
> 作者：Seyed Ali Mirheidari, [Sajjad Arshad](https://twitter.com/sajjadium), [Kaan OnarliogluAkamai Technologies](https://www.onarlioglu.com/), [Bruno Crispo ](http://disi.unitn.it/~crispo/), [Engin Kirda](https://www.ccs.neu.edu/home/ek/), [William Robertson ](https://wkr.io/) 
>
> 单位：Università di Trento, Northeastern University, Akamai Technologies
> 
> 原文：[Cached and Confused Web Cache Deception in the Wild](https://arxiv.org/abs/1912.10190)

## Abstract

​	作者第一个提出了针对**Web缓存欺骗 (WCD)攻击**的大规模测试的研究工作，量化了Alexa前5K名中的340个高知名度的站点中WCD的流行情况。作者实验表明，在WCD公开两年后，仍有许多站点的页面存在被攻击的可能。作者对于这一现象提出了建议：CDN服务供应商不要使用即插即用的技术。

<!-- more -->

## Introduction

- CDN现状：一般CDN缓存的资源包括了用户经常访问的图像，软件和文档下载，流媒体，样式表以及大型静态HTML和JavaScript文件。CDN服务提供商为Alexa Top 1K中74%的站点提供了服务，这表明CDN在Internet中发挥着核心的作用。
- WCD攻击：2017年，Gil提出了WEB缓存欺骗（WCD）攻击，该攻击可以诱使Web缓存服务器错误地存储与包含被攻击者信息的资源，攻击者在这种情况下访问该资源以实施攻击。该攻击的核心是源服务器和WEB缓存之间的路径混淆。
- 本文工作：
  - 对Alexa Top 5K中295个站点的WCD的大规模测量和分析
  - 对WCD严重性的新见解
  - 对CDN供应商进行实证分析，记录了它们的默认缓存设置和自定义机制

## Background & Related Work

本节简要概述了Web缓存欺骗（WCD）攻击的原理，攻击所利用的路径混淆技术，以及现有的WCD检测工具。

### Web缓存

在Internet上反复传输大量的Web资源是一个昂贵的过程，当面对Internet基础结构和路由问题时，网络延时问题将更加严重。为了解决这一问题，在Web资源传输的各个环节都设置了Web缓存机制。其中CDN技术（内容分发式网络）是web缓存机制的重要组成部分。它通过在全球范围内大规模部署缓存代理网络（即边缘服务器）实现。CDN的目标是使用户尽可能多从距离其最近的边缘服务器中的缓存获得请求资源。

缓存的最常见目标是静态但经常访问的资源。 其中包括静态HTML页面，脚本和样式表，图像和其他媒体文件，以及大型文档和软件下载。（非静态对象的缓存并不常见，并且与WCD攻击无关。本文没有讨论。）HTTP / 1.1规范定义了可在服务器响应中包含的Cache-Control header，以向通信路径上的所有Web缓存发信号通知如何处理传输的对象。 例如，标题“ Cache-Control：no-store”指示不应存储响应。 虽然规范指出Web缓存必须尊重这些表头，但是Web缓存技术和CDN提供程序为用户提供了配置选项，缓存结果将忽略或者覆盖标头指令。 

### Path confusion 路径混淆

路径混淆技术基于URL重写。

对于URL重写，举例来说，http://www.somehost.com/Blogs/2006/12/，经过URL重写后变为
http://www.somehost.com/Blogs.aspx?year=2006&month=12。在现在很多服务器都会将用户请求的URL进行重写，并按照其重写后的结果响应用户的请求。但是，由于Web服务器并没有告知外部其对复杂URL重写机制的结果，其可能会以不同于用户期望的方式（可能是不安全的）方式处理用户请求的URL。

对于路径混淆机制，举例来说，当用户使用example.com/index/img/pic.jpsg访问相同的Web服务时，服务器将URL重写为example.com/index.php?p1=img&p2=pic.jpg，并使用HTTP 200状态代码返回index.php的内容。值得注意的是，即使img/pic.jpg是Web服务器上不存在的资源，HTTP 200状态代码也表明将错误地URL请求已按预期成功处理。

### Web Cache Deception WEB缓存欺骗

![1](/images/2020-04-07/1.PNG)

如图所示，这是WEB缓存攻击的主要步骤。

1. 攻击者诱骗受害者访问请求为**/account.php/nonexistent.jpg**的URL。
2. 请求到达了Web服务器并被处理。 注意到，处理的资源为/account.php，而丢弃了请求对象的不存在的部分。结果，服务器会发送回成功响应，并包含了用户的私人信息。Web缓存服务器将URL解释为资源为图片的结果，并缓存网页。
3. 攻击者访问相同的URL，缓存命中，攻击者获得受害者信息。

### WCD的预防手段
研究人员发布了用于扫描和检测WCD的工具，例如，作为Burp Suite扫描仪的扩展或作为独立工具。 作者注意到这些工具是针对渗透测试的，旨在直接在测试人员的控制下对Web属性进行针对性的扫描。

## Methodology

#####  3.1 Stage 1: Measurement Setup 初始设置

- 域发现：作者使用子域名枚举工具，枚举了待测试站点的域名的子域名；
- 用户创建：作者为每个站点创建了两个账户，一个是受害者账户，另一个是攻击者账户。
- cookie收集：当作者使用每个账户访问站点时，将该账户使用的Cookie。这些cookie被保存在一个cookie罐中，以在随后的测试步骤中使用。

##### 3.2 Stage 2: Attack Surface Detection

这一部分的主要工作是将获得的子域名映射到有效的页面的URL上。这些页面的URL将在随后将进行WCD漏洞测试。

##### 3.3 Stage 3: WCD Detection

对于Stage2中发现的每个URL，作者发起了WCD攻击，攻击步骤如下：

1.  首先设计**不存在的静态资源的攻击URL**——在原始页面URL附加上/random.css请求资源。这防止普通用户巧合请求了相同的资源。
2.  作者使**受害者帐户**发起对此攻击URL的请求并记录响应。
3.  作者使**攻击者帐户**发出相同的请求，并保存响应以进行比较。
4.  最后，作者通过删除保存在攻击者cookie罐中的所有会话标识符，以**未经身份验证的用户**身份来重复攻击。 
5.  结果提取
    1.  标记提取，检查攻击者访问的URL中是否有与受害者账户有关的信息。
    2.  秘密提取，作者检查了页面对攻击者的响应中是否有秘密令牌，以及相关资源中是否有包含受害者身份的秘密信息。

## Web Cache Deception Measurement Study

### Data Collection

由于一些网站并没有使用CDN服务；且为了方便作者创建受害者与攻击者账户，作者挑选了可以使用GOOGLE OAuth登录的站点，本次实验的网站站点个数为295个。

![](/images/2020-04-07/Snipaste_2020-04-07_22-21-29.PNG)

###  Measurement Overview

1. 在295个站点中，作者识别出出了16个站点（5.4％）包含WCD漏洞。其在Alexa上的排名情况如下图所示。

2. 作者在受到WCD攻击的站点返回的HTTP标头中搜索了与CDN供应商相关的字符串。 表3显示了此项结果。 许多站点使用多个CDN解决方案。

   

3. 响应代码。 表4列出了针对易受攻击的站点观察到的HTTP响应代码的分布。 可以看到“404 NOT FOUND"为主要的响应代码。虽然只有12个站点泄漏了返回了200 OK，当作者对这些漏洞进行手动检查时，作者注意到从此类资源中泄漏了更多的PII。

   ![](/images/2020-04-07/Snipaste_2020-04-07_22-22-46.PNG)

### Vulnerabilities

![Snipaste_2020-04-07_22-23-53](/images/2020-04-07/Snipaste_2020-04-07_22-23-53.PNG)

1. **PII** 14泄漏了各种PII，包括名称，用户名，电子邮件地址和电话号码。除了这四个主要类别之外，还发现了其他多种PII类别被泄漏。 其他PII的广泛示例包括财务信息（例如帐户余额，购物历史记录）和健康信息（例如烧掉的卡路里，步数，重量）。  虽然很容易忽略此类信息，但作者注意到，上述PII可以用作高效的鱼叉式网络钓鱼攻击的基础。
2. **Security Tokens**  在16个易受攻击的站点中，有6个泄漏了会话有效的CSRF令牌，在这种情况下，即使网页部署了CSRF防御，攻击者还是可以发起CSRF攻击。其中有4个使在嵌入的JS脚本中发现，这些JS主要是用了发起HTTP请求。3个使在POST请求表单中发现的。有2个站点在GET请求中泄露了用户的信息。
3. **Authenticated    vs.    Unauthenticated    Attackers.**  作者在第三节测试时，使用了未经身份验证的用户重复发起了攻击，这种攻击仅在少数情况下失败。 这意味着WCD攻击的前提并不要求攻击者进行身份验证。 

## Variations on Path Confusion

在本节，作者首先提出了一个假设：当路径混淆技术发生变化时，可能会触发更多的WCD攻击。因此在第一次实验的基础上，时隔14个月进行了相同的实验。

在第二次实验中，作者使用了不同的路径混淆技术。为了消除之前使用账户的偏差，作者不再专门挑选GOOGLE OAuth进行身份认证的站点，而是选择了手动执行账户创建。在本次实验中，作者总共扫描了340个站点。

### Path Confusion Techniques

![](/images/2020-04-07/Snipaste_2020-04-07_22-24-17.PNG)

为了减少404的状态返回码，作者使用了一下四种新的路径混淆办法。

- **编码换行符（\ n）** Web服务器和代理通常以换行符作为停止解析URL的标志，而丢弃URL字符串的其余部分（请参见图4b）。而缓存服务器则将URL的标志解析为静态资源而缓存该网页。
- **编码分号（;）** 构造方法参见图4c，原因同上。
- **编码磅（＃） **构造方法参见图4d，原因同上。
- **编码问号（？）** 构造方法参见图4e，原因同上。

### Results

因此在第二次研究中作者发现了25个站点有页面存在WCD漏洞。在第一次实验后，作者通知了16个存在漏洞的站点。在上次16个站点，只有4个站点仍有WCD漏洞。在本次实验中，作者发现了25个易受攻击的网站，其中有20个属于先前使用295个使用Google OAuth的站点，而5个是新选择的不使用Google OAuth的网站。作者使用了Pearson的χ2检验方法证明了改变站点的身份认证机制（即不再使用Google OAuth），不会影响作者的发现。

**响应代码** 结果如图7所示。响应码为"200 OK"的页面数显著增加，这意味着改变路径混淆技术可以使攻击者能够得到受害者隐私信息的概率更大。

![](/images/2020-04-07/Snipaste_2020-04-07_22-24-36.PNG)

**漏洞**  在此实验中，作者总共确定了25个漏洞站点。表8显示了使用不同路径混淆变化检测到的易受攻击的页面，域和站点的细分。总体而言，原始的路径混淆技术已经可以扫描到68.9％的页面和14个站点，而新的路径混淆技术，然可以发现98.0％存在漏洞的页面，以及25个存在漏洞站点中的23个，这这表明增加路径混淆技术可以显著增加了成功攻击的可能性。

## Empirical Experiments

对WCD漏洞的利用取决于许多因素，例如使用的缓存技术和配置的缓存规则。 在本节中，作者提供了两个实验性经验，以演示不同的缓存设置对WCD的影响，并对流行CDN提供商的默认设置进行了探索。

### Cache Location

成功的WCD攻击需要攻击者与受害者请求一个Web缓存服务器的资源，这就要求对攻击者和受害者的地理位置有所要求。在本次实验中，受害者在波士顿访问上一次实验中存在WCD漏洞的25个站点，作者在意大利特伦托的另一台服务器上请求相同的资源。结果发现，其中25个站点中，19个站点WCD攻击失败，6个成功。

在仔细检查网络流量后，作者发现6个站点中的1个是由于在意大利区域记录了缓存未命中，然后在波士顿地区进行请求，缓存命中，从而攻击成功。这种情况表明，有些WCD管理是分层缓存模型，当这一层缓存未命中时，将请求上一层的缓存。对于其他的易受攻击的站点，可能性是这些易受攻击的站点使用了由它们的CDN提供商提供的单独的集中式服务器端缓存。

作者的实验证实了缓存位置对于一个成功的WCD攻击来说是一个约束条件，因为它涉及到一组分布式的缓存服务器。但同时也证明了当不需要操纵流量时，攻击在某些情况下是可行的。

### Cache Expiration 

 为了衡量缓存过期对WCD的影响，作者分别以1小时，6小时和1天的延迟重复了作者的攻击实验。作者发现在每种情况下，仍然存在WCD漏洞的站点个数为16、10和9个。
 这个结果表明：缓存最终会清除敏感数据，这意味着延迟时间较短的攻击更有可能成功。

### CDN Configurations

作者对CDN缓存配置进行了实验。作者在四个主要CDN提供商（Akamai，Cloudflare，Cloud-Front和Fastly）上创建了免费或试用帐户。结果表面表明，Cloudflare、CloudFront和Fastly提供了适合个人使用的免费账户，同时，它们也具有付费级别，这些级别可以解除某些限制并为高级定制提供专业的服务支持。Akamai则严格执行business-to-business的配置，由专业的服务团队驱动。

**Cacheability** 接下来作者关注了这些公司默认的缓存设置的配置。结果如下表所示，在默认情况下，Akamai和Cloudflare在做可缓存性决策时，都依赖于一个预定义的静态文件扩展名列表(例如.jpg、.css等)。CloudFront和Fastly则采用了更具“攻击性”的缓存策略：在没有Cache-Control标头的情况下，所有对象都用一个默认的生存时间值进行缓存。 

![](/images/2020-04-07/Snipaste_2020-04-07_22-25-03.PNG)

### Lessons Learned

作者在本节中提供的经验性的结果表明，正确配置Web缓存不是一件容易的事，与发动攻击相比，检测和修复WCD漏洞的复杂性高得多。如上结果所示，主要的CDN供应商在他们的默认配置中没有做出符合RFC的缓存决策。虽然CDNs可能是Internet基础设施的一个组成部分，但WCD攻击会影响所有web缓存技术。

对于外部安全研究人员来说，这个挑战更大。正如在之前的实验讨论的，对于攻击者来说，CDN网络是一个黑匣子，不易搞清楚其内部的运行机制，反过来看，攻击者基本上不受这种复杂性的影响，因为在很多情况下无需了解缓存结构就可以成功发起一次攻击。

## 总结

本文大规模测试了WCD漏洞现状，并对测试的结果进行了探讨。

CDN是近年来应用较广的提升web性能的技术，今年关于CDN安全方面的讨论也较多——USENIX 2020已有两篇讨论了CDN的安全问题（不过切入点非常不同），NDSS 2020有一篇。在本文中，作者强调了现在针对WCD漏洞的检测手段较少，后期读者可以针对这个问题展开研究。

在测试方法上，我认为作者改变（改进）了变量，并在不同时间内进行重复性实验是一个亮点，在以后的测试工作中也可以使用这一方法来观测时间对于实验结果的影响。