---
id: 132
title: 整洁代码-类与系统设计
date: 2018-08-23T21:24:04+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=132
permalink: /2018/08/cleancode-class-and-system-design/
categories:
  - program
tags:
  - 开发技术
  - 笔记
  - 规范
format: aside
---
为了写出整洁的代码，不仅仅需要函数和代码行块上的优化，同时需要考虑一些更高层次的设计。

### 1&#46; 类

类的组织，尽量按照公共静态常量，私有静态变量以及私有实体变量，变量后面跟随函数列表。私有工具函数尽量保持与调用者在一起，保证阅读的简洁。

类应该短小，类名能够准确描述定义。含糊的命名，过长的命名有可能代表了设计的类有问题。

  * 符合单一权责原则（SRP） : 仅有一个加以修改的理由（代表了功能的单一）， 系统应该由更多小的类组成，每个小的类完成一个职责，并与其它一起协同达到期望的系统行为。

  * 内聚： 类中的方法和变量之间的互相依赖关系，组合成一个逻辑整体，高内聚（代表了类中的实体都是类所需要的，反之则代表可能根本不需要该实体变量，也就没有继续保留在类的必要）

> 对于类的重构，优化的方式需要考虑将原来类中的较大的函数拆解为多个小的函数，小的函数再考虑单一职责原则，是否有必要独立成一个单独的类进行逻辑处理。

在对系统进行修改或者添加特性时候尽可能少的减少对于原有代码的逻辑修改，开放闭合原则： 对扩展的开放，和对于修改的封闭。

#### 依赖倒置原则

系统解耦，隔离测试，**依赖倒置原则**： 类应该依赖于抽象而不是依赖于具体实现， 比如测试时候的Mock类，本身提供必要的接口函数来满足调用，但是又去除了本身的细节，使得测试更加容易。

就像一个执行任务的调度器一样，假设所有任务都应该具有Run()函数接口，当调度器调用的时候不需要知道内部的设计是如何工作的，只需要调用该函数即可，因此是面向接口抽象的编程，而不是面向实现的编程，反例就是： 调度器嵌入到每一个任务中去直接管理任务的执行。

### 2 系统设计

软件团队也应该像构造城市一样，演化出恰当的抽象等级和模块，各司其职。保持系统层面的整洁。

构建和使用的分开，延迟初始化和赋值，减少初始化的时间，但是会造成测试时候的不便，无法确定是否对象在恰当的时间和调用时候初始化已经完全。 DI 依赖注入, 借用Stockoverflow上的解释如下：

> When you go and get things out of the refrigerator for yourself, you can cause problems. You might leave the door open, you might get something Mommy or Daddy doesn&#8217;t want you to have. You might even be looking for something we don&#8217;t even have or which has expired. What you should be doing is stating a need, &#8220;I need something to drink with lunch,&#8221; and then we will make sure you have something when you sit down to eat.

基本的意思就是说，不同的示例尽量只是做好自己的事情，不要尝试自己去创造一些对象， 把这些创造的过程交给其他的对象自己去做。 在类创建的过程中，假如一个类需要在初始化的时候或者过程中去创造另一个类去执行操作，应该考虑以传递的方式去实现，而不是自己去创建， 这样测试的时候也更加简单一些。

    public SomeClass() {
        myObject = Factory.getObject();
    }
    
    # DI的方式
    public SomeClass (MyClass myObject) {
        this.myObject = myObject;
    }
    

> 拆分构建和使用过程，会使得代码的逻辑更加简单。测试更容易。

##### 代理模式

代理模式使得对于对象的访问增加不同的控制条件，或者实现对于原来的对象增加新的功能等。代理模式的使用，以下为python的代理模式使用例子

    """
    Proxy pattern example.
    """
    from abc import ABCMeta, abstractmethod
    
    
    NOT_IMPLEMENTED = "You should implement this."
    
    
    class AbstractCar:
        __metaclass__ = ABCMeta
    
        @abstractmethod
        def drive(self):
            raise NotImplementedError(NOT_IMPLEMENTED)
    
    
    class Car(AbstractCar):
        def drive(self):
            print("Car has been driven!")
    
    
    class Driver(object):
        def __init__(self, age):
            self.age = age
    
    
    class ProxyCar(AbstractCar):
        def __init__(self, driver):
            self.car = Car()
            self.driver = driver
    
        def drive(self):
            if self.driver.age <= 16:
                print("Sorry, the driver is too young to drive.")
            else:
                self.car.drive()
    
    
    driver = Driver(16)
    car = ProxyCar(driver)
    car.drive()
    
    driver = Driver(25)
    car = ProxyCar(driver)
    car.drive()
    

城市建设也不是一气呵成，而是经历不同阶段的重新调整和设计， 软件架构也是如此，迭代重构和增量敏捷。但是又有不同之处，比如建筑设计上面一般都是按照BDUF（宏观设计优先）的方式，软件开发的时候，当然也需要考虑全局设计和架构等方面，但是软件如果设计得当，松散耦合就会很方便的进行重构， 开发同样需要具有随机应变的能力。 测试驱动开发和整洁代码保证整个过程的顺利实施。

### 3 迭代开发

Ken Beck的简单设计四条规则：

##### 1&#46; 运行所有测试

不可测试的系统，无法得到验证，也就不能得到部署. 应该全面测试并持续通过所有的测试的系统，才算是可测试的系统。应该尽量减少耦合度，编写尊徐SRP和DIP的类

编写足够多的测试。

消除对于代码变更的恐惧， 也就能够获得递增式的代码重构的能力。

##### 2&#46; 不可重复

去除重复的代码。不仅要考虑减少代码的重复，也要考虑对于代码中的重复的复用，是否有必要单独的抽象出类来处理问题。利用模板的继承的方式减少重复代码。

  1. 表达程序员的意图

代码简洁，函数命名规范，短小的类和函数设计。 使用一些设计模式的类方式 使用测试，并写好测试，提供别人参考实例。 不断的优化代码

> 做好代码的长期维护的准备

  1. 尽可能减少类和方法的数量

防止教条主义