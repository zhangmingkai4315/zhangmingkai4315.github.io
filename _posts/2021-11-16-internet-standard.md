---
layout: post
title: 互联网标准规范
date: 2018-11-16T11:32:41+00:00
author: mingkai
permalink: /2018/11/rfc-internet-standard
categories:
  - rfc
---

**互联网标准**（InternetStandard）是用来定义使用在互联网上使用的某些技术和方法的一种业内标准, 由IETF(Internet Engineering Task Force)创建并发布。 这些标准一般都需要作为互联网草案（Internet Draft）提交给IETF, 后期经过审批可以成为RFC(Request for Comments), 最终才有可能成为互联网标准。

一个互联网标准首先必须是技术上成熟且有用的标准， IETF同时定义了**建议标准（Proposed Standard）**作为一个成熟度尚且不够但是被广泛审核过的规范；**标准草案（Draft Standard）**在2011年不再使用了，之前作为建议标准和标准之间的一个中间阶段。

所有的定义均可以在[RFC2026](https://tools.ietf.org/html/rfc2026) 互联网标准流程-V3中查看到。其中给出的定义如下：

> In general, an Internet Standard is a specification that is stable and well-understood, is technically competent, has multiple, independent, and interoperable implementations with substantial operational experience, enjoys significant public support, and is recognizably useful in some or all parts of the Internet.
>
> 一般而言，互联网标准是一种稳定且易于理解，技术导向且具有多个，独立以及丰富的实施运行经验，得到很多相关专家的支持，并且对于部分或者整个互联网有用的规范标准。



#### 标准的形成

一个互联网的标准被记录在一个RFC或者多个RFC中。一个规范需要经历多个步骤才能最终成为互联网标准。 首先需要成为一个互联网草案（Internet Draft）, 经过RFC审核人员(RFC Editor)多次审核编辑后成为RFC,并标记为**建议标准（Proposed Standard）** . 当建议标准的成熟度到达一定程度后，RFC会被提升为一个内部的标准，并增加一个序列号，比如1034， 1035这些序列号。这些阶段都被称之为标准追踪（Standards Track）阶段. 只有IETF可以审议通过标准追踪阶段的RFC

> 最终成为一个标准的两个阶段：建议标准和互联网标准，整个的流程成为标准追踪流程

- 建议标准：如果一个RFC进入建议（提议）阶段，也就是第一个阶段，一些组织或者机构可以考虑是否应用到该建议到实际环境中，后期符合RFC6410中的条件（两个独立的实施，且广泛的使用，没有描述性错误等），这个RFC才能成为互联网标准。 由于可以部署使用，因此建议标准可以在出现任何问题或者最佳实施后被修改。很多现有的技术实施其实都还是处以建议标准阶段，并且已经作为稳定的协议被广泛的使用。
- 草案标准：已经被划分到建议标准阶段
- 互联网标准：该阶段被任何为是技术足够成熟且足够重要和有用的一些规范标准，比如IP协议的一些标准RFC. 对于普通的RFC文档，如果发生变化，重新提交会被分配一个新的RFC号码。但是当RFC成为互联网标准后（STD）会被分配一个STD号码，并且保留原来的RFC号码。当一个互联网标准发生变更的时候，这个号码不会改变，但是会指向引用其他的RFC。 比如RFC3700作为一个STD标准，并且在2018年做了修改，并由RFC5000替换，原来的3700被标记为历史(Historic)状态,RFC5000变成STD.



所有的规范可以在[标准列表](https://www.rfc-editor.org/standards)查询到, 涉及DNS的标准如下

- STD13 RFC1034  域名的概念和设施
- STD13 RFC1035  域名实施和规范
- STD74 RFC5011 自动更新安全信任锚
- STD75 RFC6891 DNS扩展机制EDNS0
- STD89 RFC4443 DNS扩展支持IPv6
- ... 

另外的很多处于建议标准阶段，比如下面的一些内容，但是尽管没有成为标准，但是也已经被广泛使用：

- RFC1995 实施区传输
- RFC2136 DNS动态更新
- RFC2308 否定缓存
- ...

