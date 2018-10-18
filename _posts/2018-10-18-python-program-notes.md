---
id: 41
title: Python编程记录-类型与类
date: 2018-10-18T11:32:41+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=41
permalink: /2018/07/python-program-notes/
categories:
  - python
tags:
  - python
---
Python3.0发布2008年12月，PEP404定义了取消发布Python2.8版本的计划，Python的很多改进都依赖于特定领域的需求驱动，比如web方向的高并发逐步再Python3版本中得到实现。 PEP文档定义了很多Python程序语言的信息，代码风格，指导意见以及对于功能的设计等，PEP0包含了一个所有PEP的索引内容。

Python2到3版本的迁移可能需要付出比较大的成本，直接使用2to3工具可能会导致转换后的代码还不如之前的性能， 使用一些优秀的软件架构设计方法可以逐步的帮助实现目标，比如拆分项目，微服务架构设计等。

#### 1. Python2和Python3的主要区别

区别分为三大类：
1. 语法变化，删除或者修改一些语法元素或者新增一些语法
- 使用print不再是一个语句，而是一个函数，必须加括号
- 捕获异常改为```except exc as var```
- 不能再使用<>符号，直接使用!=
- from module import * 现在只能用在模块中， 函数中不能再被使用
- from .module import name 是相对导入的唯一方式
- 整数除法将返回浮点数，如果需要返回整数使用//符号，比如```5.0//2.0 == 2.0```
- sorted函数参数发生变化

2. 标准库的变化
部分标准库进行了重构和声明，比如urlparser迁移到了urllib.parse中使用

3. 数据类型和集合变化
Python3版本中的字符串默认为Unicode, 字节bytes需要加入b或者B的前缀，不再支持使用u来进行unicode转码（3.3版本后恢复，但无效果，仅仅为了兼容保留）

#### 2. 兼容的处理方式

使用```from __future__ import * ```来定义在python2版本中支持新的版本功能，比如下面，使用默认的unicode编码：

```
from __future__ import unicode_literals
type("foo")  # unicode
```
future模块还包含了新增的除法标示(division)，以及绝对导入(absolute_import)的使用，print_function的使用等

另外一些情况使用sys.version_info < (3,0,0)来进行判断分别执行也可以实现兼容，但代码量可能会比较大。

#### 3. 不同版本的Python

1. CPython

使用最多的Python版本，使用C语言实现，其他版本的功能实现一般落后与该版本，是主流的Python版本。

2. StacklessPython

引入微线程概念管理，而不是像CPython中的内核切换，并发性能更好。内核切换功能实现的库为greenlet，多个软件库中都有实现。

3. Jython  使用java垃圾回收，而不是引用计数，取消了GIL，多线程中可以充分使用内核。Python模块中可以调用Java库实现，但C语言的Python扩展无法正常运行
4. IronPython， 没有GIL, 可以集成C#和其他.net语言编写代码，使用Silverlight， 可以在主流浏览器使用。
5. PyPy Python解释器使用Python编写。运行速度较快，使用垃圾回收，集成JIT编译器，运行效率高


#### 4. Python内置类型

Python3版本中只有一种能够保存文本信息的数据类型，就是str(字符串),保存的是unicode编码字符，属于不可变的序列。 Python2版本中str标示的是字节字符串，这在python3中使用bytes来标示处理（字节属于x[0,256]范围的整数）

在python3中如果要定义字节字符串需要使用b或者B开头，比如下面

```python
>>> print(bytes([102,111,111]))
b'foo'
>>> list(B"foo")
[102, 111, 111]
```

字符串拼接处理的时候，依赖于拼接的次数来决定使用直接+拼接还是使用join方式计算。性能可能差距比较大，但是如果较少的时候使用join反而运行效率较低。

> Python3版本中的str如果需要保存到磁盘和传输到网络中需要执行encode编码为二进制进行处理。


python列表实现为长度可变的数组，而非链表，修改列表长度可能导致数组长度的改变，在普通链表中实现的比如插入元素或者删除元素，在数组中可能需要较高的福再度O(n).如果需要使用链表则，可以使用collections库中的deque双向链表结构。

使用列表推导，enumerate， zip,参数捕获来获取更高效简单的代码：

```python
a= [i for i in range(10) if (i % 2) == 0]
print(a)
# [0, 2, 4, 6, 8]
for i, element in enumerate(['a','b','c']):
  print(i,element)
# 0 a
# 1 b
# 2 c
for i in zip([1,2,3],[4,5,6]):
  print(i)
# (1, 4)
# (2, 5)
# (3, 6)
a, *b, c = [1,2,3,4,2,5,4,3]
print(a,b,c)
# 1 [2, 3, 4, 2, 5, 4] 3
```

对于dict类型现在的keys,values,items方法不再返回list，而是返回一个dict_keys, dict_values和dict_items视图对象。可迭代对象

dict类型的键必须为可hash的对象，所有不可变对象均为可hash的对象，可变类型（列表字典和集合）是不可hash的对象，不能作为字典的键，使用__hash__和__eq__两个方法来支持可hash类型协议。

集合实现为带有空值的字典，只有键才是实际的集合元素，删除添加和检查元素平均复杂度为O(1), 而最坏复杂度为O(n), 集合分为两种一种为set另一种为frozenset，是不可变的可hash的无序集合。Set本身是可变的，无序的有限的集合。

#### 5. 迭代器

自定义的类满足迭代器协议的可以利用列表推倒获得所有迭代的值，迭代器协议中__next__注意要在超出限制的时候抛出StopIteration异常，__iter__返回当前迭代器实例。

```python
class CountDown:
  def __init__(self,number):
    self.number = number
  def __next__(self):
    if self.number <=0:
      raise StopIteration
    self.number -= 1
    return self.number
  
  def __iter__(self):
    return self

print([i for i in CountDown(4)])
```

使用generator来获得一个迭代器，可以减少内存使用，而且不必提供函数停止的方法，比如下面实现的faboacci函数

```python
def fab():
  a, b = 0, 1 
  while True:
    yield a
    a, b = b, a+b
    if a >1000:
      return
print([i for i in fab()])
```

#### 6. 装饰器

装饰器通常是一个命名的对象，在被装饰函数调用时接受到单一的参数，并返回一个可调用的对象
```python
def log(my_func):
  def func(*args, **kwargs):
    print("log start")
    my_func(*args, **kwargs)
    print("log stop")
  return func
@log
def show_name(name):
  print("this is a {} function".format(name))
show_name("test")
```

上面的例子还可以使用类来实现，如下：

```python
class LogDecorater:
  def __init__(self,func):
    self.func = func
  
  def __call__(self, *args, **kwargs):
    print("log start")
    self.func(*args,**kwargs)
    print("log stop")    
```

带参数的装饰器, 注意其中的repeat必须使用括号，否则会报错

```python
def repeat(number = 10):
  def decorate(func):
    def wrapper(*args, **kwargs):
      for i in range(number):
        result = func(*args,**kwargs)
      return result
    return wrapper
  return decorate

@repeat()
def show_welcome():
  print("welcome")
```

使用functools模块中的wraps函数，我们可以保留原始的函数中的一些元数据：

```python
from functools import wraps

def preserving_decorator(function):
  @wraps(function)
  def wrapped(*args,**kwargs):
    """this is inner doc"""
    return function(*args,**kwargs)
  return wrapped

@preserving_decorator
def func_with_doc():
  """Important doc"""
  pass

print(func_with_doc.__name__)
print(func_with_doc.__doc__)
# func_with_doc
# Important doc
```

下面是一个缓存的实例，用装饰器的方式处理缓存，在多个框架中都有实现（django, flask...）

使用for ... else 来执行一些正常循环结束后才会做的操作

#### 7.子类化内置类型

使用内置的list,dict可以作为类的子类来实现，比如下面的实现一个打印文件夹的方法, 继承自list类，获取到所有list的方法和属性，并实现自己的方法。

```python
class Folder(list):
  def __init__(self, name):
    self.name = name
  
  def dir(self, nesting=0):
    offset = " " * nesting
    print("%s%s/" % (offset,self.name))
    for element in self:
      if hasattr(element, 'dir'):
        element.dir(nesting+1)
      else:
        print("%s %s" % (offset, element))
```

> Python2中的类继承如果没有定义父类，则不会默认继承自Object，这些称之为旧类继承, 新式类则继承必须显式继承object.python3中则缺省所有都继承自Object,因此为了兼容性，还需要同时加上（object）的继承，否则在python2中定义为旧类'

#### 8.MRO

Python2和Python3在多重继承上的执行顺序是不同的，相同的代码在执行的时候结构也不同：

比如下面的代码及在python3环境下的执行结果：

```python
class Base:
  def method(self):
    print("Base")

class F1(Base):
  pass
  
class F2(Base):
  def method(self):
    print("F2")

class C1(F1,F2):
  pass

c = C1()
print(C1.mro())
c.method()

# [<class '__main__.C1'>, <class '__main__.F1'>, <class '__main__.F2'>, <class '__main__.Base'>, <class 'object'>]
# F2
```

在python2中执行，无法打印mro（旧类，没有继承自object）,采用从左到右，深度优先的方式进行查找，返回Base

注意事项：
1. 避免使用多重继承
2. 不要混合使用super和直接调用```A.__init__```这样的方式调用。
3. Python3中也最好使用显式的继承object
4. 查看对象的mro来检查可能会造成的继承问题。


#### 9.属性访问

对于想要隐藏的对象，使用```__name```来作为属性名称，取得属性的时候会报错误，因为实际存储的为类名称和属性名的组合, 如下面所示：

```
class Info(object):
  def __init__(self,name):
    self.__description = "infomation class"
    self.name = name

i = Info("Mike")
print(i.name)
print(i._Info__description)
```

使用属性访问可以创造延迟求值的场景，并减少实际的调用过程中内存的消耗，比如下面的代码：


```python
class InitOnAccess(object):
  def __init__(self, klass, *args, **kw):
    self.klass = klass
    self.args = args
    self.kw = kw
    self.__initialized = False

  def __get__(self, instance, owner):
    if self.__initialized == True:
      print("cached")
    else:
      print("Initialized")
      result = self.klass(*self.args, **self.kw)
      print(result)
      self.__initialized = True

class MyClass:
  lazily = InitOnAccess(list, "arguments")

m = MyClass()
m.lazily
m.lazily
# Initialized
# ['a', 'r', 'g', 'u', 'm', 'e', 'n', 't', 's']
# cached
```

使用property来定义函数转换为属性访问的方式，对于get和set的方式不同，比如下面的代码中：

```python
class Rectangle:
  def __init__(self, x1,y1,x2,y2):
    self.x1, self.x2 = x1,x2;
    self.y1,self.y2 = y1,y2
  
  @property
  def width(self):
    return self.x2-self.x1

  @width.setter
  def width(self, value):
    self.x2 = self.x1+value

r = Rectangle(10,10,20,20)
print(r.width)
r.width=20
print(r.width)
```


#### 10. 槽的用法

使用槽可以预先定义一个对象内部的属性参数列表，并限制动态的增加行为，对于需要实例化大量类对象的操作，使用槽可以显著的节省内存使用空间。不需要为每个实例创建__dict__方法。

```python
class Info(object):
  __slots__=['name','age']
  def __init__(self,name,age=10):
    self.name = name;
    self.age = age


i = Info("Mike",'30')
i.NotExist = True # AttributeError

```

#### 11 元编程

元编程主要分为两个大类：

1. 专注于语言对基本元素（函数，类和类型）的内省的能力和对其实时修改和创建的能力。最典型的是装饰器，允许增加功能和修改已定义的函数等。

```python
def short_description(cls):
  cls.__repr__ = lambda self: super(cls,self).__repr__()[:10]
  return cls

@short_description
class InfoClassWithLongName(object):
  pass

print(InfoClassWithLongName())
# <__main__.
```

另外使用__new__可以在初始化之前执行一些操作，new操作要比init操作更早，因此此时尚未实例化任何对象， 函数返回应该是该类的实例，但也可也不是，当不是的时候，init函数将不会被执行。

```python
class CounterClass(object):
  counter = 0
  def __new__(cls, *args, **kw):
    instance = super().__new__(cls)
    instance.number = cls.counter
    cls.counter += 1
    return instance

c1 = CounterClass()
print(c1.number)  # 0
c2 = CounterClass()
print(c2.number)  # 1
```

2. 类的特殊用法，允许修改类实例的创建过程，典型的使用工具是元类。

元类用来定义其他类的一种类，所有对象实例的类也属于对象，默认类定义的基类都是内置的type类，比如下面的代码,与直接定义class方式相同，只不过python里面所有的class默认的元类都是type：
```
def method(cls):
  print("method called")
  return cls

klass = type("MyClass",(object,),{"method":method})
k = klass()
print(k.method())
# method called
# <__main__.MyClass object at 0x1040947f0>
```

python2和3在定义元类的时候是不同的方式：

```python
class Python3MetaClass(metaclass=type):
  pass

class Python2MetaClass(object):
  __metaclass__ = type
```

动态代码生成和运行时修改，可以使用eval和exec函数执行，但是一定要注意安全访问，不要和不安全的输入一起使用。使用代码动态生成另一种方式是ast模块，该模块可以用来将python代码转换为抽象语法树，在传递给compile调用前进行修改。（参考项目：MacroPy）

