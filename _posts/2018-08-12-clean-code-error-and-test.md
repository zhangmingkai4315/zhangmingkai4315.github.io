---
id: 122
title: '《代码整洁之道》 &#8211; 错误处理及测试相关'
date: 2018-08-12T14:31:31+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=122
permalink: /2018/08/clean-code-error-and-test/
categories:
  - program
tags:
  - 开发技术
  - 笔记
  - 规范
format: aside
---
作者在此部分内容中对于错误处理方面给出了很多好的建议，比如使用异常而不是返回码，这样防止每次调用后都去检查代码的错误标示（主要担心忘记编写检查），但是比如像golang 语言中还是有一定的语言层次的编写规范，如下面：

    func NewHandler(handler Handler, err error){
        ... 
        return handler, nil
    }
    
    

大部分函数在执行的时候都会加入err返回，这样如果出错的话该位置会出现err!=nil的情况。 因此有些建议还是要结合实际的语言特性和要求进行使用。

而针对使用try-catch-finally的使用，也是在golang中被panic, recover defer机制所替代，使用场景基本一致，finally和defer都可以保证程序执行不管是否异常下都会被执行。

#### 可控异常的使用

Java支持可控异常，即异常在函数中抛出， 如果调用者对于抛出的异常未处理则无法被编译成功。 好处是明确了异常的类型，告诉调用者应该处理该异常，强制实现该规范。但是代价就是，每一个底层函数在修改自己的异常类型的时候都可能需要修改整个底层到高端的调用链，违反了开放闭合原则。

    try
    {
       method();
    }
    catch (IOException ioe)
    {
       System.out.println("I/O failure");
    }
    
    // ...
    
    void method() throws IOException
    {
       throw new IOException("some text");
    }
    

#### 封装调用

使用第三方库时，有时候做一些封装，可能会使得程序更加简单和舒服，将每一次调用可能需要处理的异常或者操作封装为一个简单的方法，进行处理。减少重复代码的使用

封装代码的时候，使用一些特例模式，将特殊情况或者异常封装到类里面，这样使得外面在调用的时候不用太多关心细节信息。

#### 特例模式

引用获取用户的例子来解释特例模式： 当我们有一个方法getUserById（）用来传递用户的ID并返回一个User实例。假如我们传递的id后获取不到该用户，可能是id不存在或者是该id已禁用，如何处理？一般处理如下：

  * 抛出异常
  * 返回一个null或者None等代表不存在
  * 返回一个特殊的用户实例

抛出异常的方式，简单粗暴，可能会终止程序的执行或者流程的执行中断，并且在某些程序语言中造成性能的消耗也比较大。返回null或者None等不存在对象，简单但是可能无法让人理解到底是哪里出了错误，定位错误和后续的处理都比较麻烦，最好不要使用。第三种方式使用特例模式则创建一个特殊的用户实例比如MissingUser或者UnAuthorizedUser等等，继续流程执行，直到某些验证点进行检查并执行不同的业务逻辑。 比如flask在登入的时候如果用户正常认证成功返回一个User对象，而未认证则返回的是一个UnAuthoriedUser对象，该对象继承了与Users相似的方法，后面处理的时候根据某些属性或者方法来进行判断不同用户的类型。

> 尽量不要在函数中返回null，造成所有调用都需要检查返回值的类型才能安全执行所有代码。
> 
> 尽量不要传递null值给函数, 导致函数判断的负担，只要时刻记住不要传递null, 一旦出现在参数列表中，则代表程序出现了问题。

#### 边界

程序在编写的时候会经常集成一些第三方的库或者其他的开源项目， 使用的时候要衡量原来接口的暴露是否会造成任何的使用问题。 比如我们直接使用Map类型存储，但是不希望程序调用者直接调用clean方法去删除所有的对象，又或者我们在使用Map的时候会大量的进行类型的转换，但是这些转换可以被封装起来，放置在一个独立包装的类里面。

    public class Sensors {
        private Map sensor = new HashMap()
    
        public Sensor getByID(String id){
            return (Sensor) sensors.get(id);
       }
    }
    

保证边界的最小权限，防止用户过度使用公共API接口直接调用。

同时在集成一些第三方的代码或者库的时候，可以尝试学习型测试的方法，不要在线上测试新的东西，通过编写测试来理解和学习第三方代码。核对结果检查对于代码的理解程度。

学习型测试的另一个好处是可以很容易的检查升级代码对于当前使用是否仍执行是ok的，因为我们在使用原来的功能的时候已经进行了代码的测试实例的编写，通过重新执行获得新版本的兼容性。

有两个团队分别进行前后端系统开发，有时候节奏不会总是一致，对于一些尚未存在的代码可以模拟实现一个接口，一旦未来有了具体的实现后，可以通过编写适配器的方式来实现代码的转换。

通过独立封装数据结构或者第三方库，或者使用适配器模式来完成集成，提升代码的质量，这样一旦发生变化，减少需要改动的代码量。

#### 测试

编写测试，确保每个犄角旮旯的代码都能执行成功。 将代码和操作系统隔离，编写一些伪造的函数，使用步进函数加速时间等等方法来改善测试

TDD测试的三定律

  * 编写测试代码，再编写生产代码
  * 只可编写刚好无法通过的单元测试，不能编译也算不通过。
  * 只可编写刚好足已通过的生产代码

测试代码足以匹敌生产代码总量

测试代码同样比较重要，因为测试代码要随着生产代码一起发生变化，后期维护成本与生产代码相同同。 同时代码质量和编写格式要求也要与生产代码保持一致（优秀）

> 单元测试 让代码保持了可扩展，可维护和可复用，有了测试就不会担心对于代码的修改，没有测试每次修改都可能带来问题和缺陷。测试覆盖率越高，信心也就越大。

测试代码要满足的重要特性之一就是：**可读性** ，要保持明确，简介和足够的表达力。

测试的流程主要以下三个部分组成，简单且易于理解的测试，维护起来更加方便。

  * 构造
  * 运行
  * 检查

每个测试一个断言，可能过于苛刻，但是简洁的测试使得测试函数更加易于理解。又或者我们使用每个测试一个概念的方法，来测试对于一个问题的处理是否通过。

#### 测试的FIRST法则

整洁测试的FIRST法则包含五个规则：

  * 快速 Fast 测试应该尽快执行结束，快速的测试也就会运行的频率更多，发现问题也就越及时。
  * 独立 Independent 每个测试应该互相独立， 单独运行，相互依赖的测试使得执行和诊断都比较困难。
  * 可重复 Repeatable 任何环境中重复测试， 测试环境生产环境以及其他的环境下都可以直接运行而不报错。
  * 自验证 Self-Validating 输出应该简单直接，告诉你是否已经通过测试，而不是自己去对比任何文件或者输出进行比对。
  * 及时 Timely 生产代码写完后再去写测试可能更加困难，你会发现代码本身难以测试，或者无法设计出可测试的代码。因此尽量使用TDD的方式进行编写代码