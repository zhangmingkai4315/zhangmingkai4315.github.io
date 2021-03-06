---
id: 50
guid: http://mikezhang.cc/?p=50
layout: post
title: 软件开发的十二因素
date: 2018-10-23T00:00:00+00:00
author: mingkai
permalink: /2018/10/software-design-12-factors/
categories: [program]
---

本篇文章介绍现在流行的应用开发中会涉及到的主要的12个因素，其中每一部分内容又会涉及很多扩展的内容，比如选择合适的工具和方法来满足构建需求。十二因素也为开发应用程序提供了一定的构建方法指导。

#### 1. 基准代码

开发中我们使用的git等版本管理工具，保留一份用于追钟代码修改的代码基准，基准代码于应用之间是一一对应，多个基准代码不能称为一个应用。如果多个应用共享一个基准代码的时候，考虑用共享独立的库。对于共享一份基准代码，我们在部署的时候仍可以借助于分支的方式进行多版本的部署，比如开发版本和测试版本的部署。

#### 2. 依赖

使用依赖的时候，一定要隔离系统的依赖，通过建立虚拟环境并建立完整的依赖声明清单来进行依赖的隔离。依赖声明和依赖隔离必须一起使用，否则不满足于12因素规范。也不要调用一些系统工具比如curl或者imageMagick.

#### 3. 配置

在环境中存储配置，不同的部署可能都会涉及不同的配置差异，比如缓存，数据库的配置等等。代码和配置需要严格的分离，判断基准代码是否可以立刻开源, 如果可以则代表其配置排除在代码之外。另外推荐在环境变量中存储配置，这样可以在部署的时候声明部署环境变量，代码不需要任何的改动。如果使用组合的方式比如，production,test和development 来定义不同的配置文件，则可能导致组合的固定，不灵活。

#### 4. 后端服务

后端服务指的是运行所需要的网络调用的各种服务，比如数据库，缓存系统或者邮件系统 ，甚至是调用的第三方发布和管理的服务。可以把这些都作为资源，这些资源的管理尽量不要修改任何的代码，比如把本地的mysql换成一些第三方的服务。另外资源及部署也要保持一个松散耦合，增加可扩展性。

#### 5. 构建，发布和运行

严格区分构建，发布和运行三个步骤，比如不允许直接修改运行状态的代码，每一个发布版本都需要对应一个唯一的发布ID，一旦发布就不能更改，任何的变动都需要产生一个新的发布版本。构建阶段一般需要人为参与，发现问题能够及时的追踪和定位，而发布和运行则应该保持一个自动化的过程，减少操作的失误。

#### 6. 进程

应用进程保持无状态和无共享，任何需要持久化的数据都需要存储在后端服务内。一些系统依赖于粘性session，指将用户session中的数据缓存在某些进程的内存中，并将同一个用户的后续请求路由到同一个进程，这种方式是在12因素中极力反对的，session中的数据应该保持在redis或者memcached中，并设置过期时间。

#### 7. 端口绑定

12因素要求应用完全自我加载，而不依赖于任何网络服务器就可以创建一个面向网络的服务。比如http://localhost:5000/ 访问应用，一个应用可以将另外一个应用作为后端服务，建立起微服务的框架结构。

#### 8. 并发

在12因素的应用设计中，进程为一等公民。web进程和后台进程（交由worker进程）分别负责不同职责，应用的进程具备无共享，水平分区的特性保证了并发的简单和稳妥。应用的进程不需要守护进程或者其他方式，一般借助于进程管理器比如systemd或者分布式的进程管理方式来管理进程和请求。

#### 9. 易处理

快速启动和优雅终止，易于处理保证了瞬间的开启和停止，进程应该追求最小启动时间，保证更小的启动时间和更敏捷的发布扩展。 一旦程序接收到终止信号，就可以保证程序的优雅终止。对于worker的终止，应该将未完成的任务退回到队列中，并且在系统故障的时候能够处理，保持程序的健壮性。比如beanstalkd.


#### 10. 开发环境和线上环境等价

缩小与线上环境的差异。减少差异也就会减少了部署的间隔，开发人员和运维人员也可以共享知识，甚至可以是相同的人。使用一些适配器来完成不同的后端服务的聚合，比如使用ActiveRecord来适配MySQL和SQLite，Celery来适配队列。但是适配仍旧由一定的差异，可能导致运行时候的兼容性，所以尽量使用docker或者vagrant来部署在本地开发

#### 11. 日志

使用一些开源工具自动的去处理日志事件流，并统一存储在Hapdoop或者hive这样的通用数据存储系统中，便于后期的查询和过滤，另外通过可视化的方式将一些数据展示在前端更直观一些，并设置一些告警门限，自动触发告警操作。

#### 12. 管理进程

一次性执行的进程和常驻进程应该使用同样的环境和配置以及程序代码。这样在执行的时候可以很方便的管理进程的状态和执行一些操作，比如数据库的迁移和状态监测。


更多的内容，可以参考[官方网站](https://12factor.net)，网站提供了大量的框架和环境的配置工具示例。同时要明白的是规则并不是一成不变的。随着技术的不断发展，新的构建方式可能会替代现有的方式。