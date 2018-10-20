---
id: 41
layout: post
title: Python包管理
date: 2018-10-20T11:32:41+00:00
author: mingkai
guid: http://mikezhang.cc/?p=41
permalink: /2018/10/python-package-management/
categories:
  - python
tags:
  - python
---

#### 1.编写程序包

为了实现python的打包，我们首先需要编写一个简单的python程序，下面实现了一个动物的类及两个基本的方法，我们将这个程序打包一下，用于其他程序使用或者上传到pypi上提供给别人使用。程序写入animal/\_\_init\_\_.py文件中。这里我们故意的引入了一个第三方包，用来执行一些简单的验证过程，仅仅为了演示第三方包如何处理的过程。

```python
import json
from validator import Range,validate

class ValidationError(Exception):
  pass

class Animal(object):
  def __init__(self,name,winds=0,legs=4):
    self.name = name
    self.winds = winds
    self.legs = legs

    validator = {
      "winds" : [Range(0,10)],
      "legs":[Range(0,40)]  
    }
    result = validate(validator,self.__dict__)
    if result.valid == False:
      validate_describe = json.dumps(result.errors)
      raise ValidationError(validate_describe)

  def __repr__(self):
    return "Animal<{}>".format(self.name)
  
  def has_winds(self):
    return self.winds>0
  
  def legs_number(self):
    return self.legs

```

同时需要创建一个setup.py的文件，用于实现打包的元数据定义和打包管理。这个文件主要依赖于setuptools库来实现。

```python
from setuptools import setup
import os
# from distutils.core import setup
with open("README.md","r") as readme:
      long_description = readme.read()

def strip_commands(l):
      return l.split("#",1)[0].strip()

def req(*f):
      require_file = os.path.join(os.getcwd(),*f)
      if not os.path.exists(require_file):
            return ""
      return list(filter(
            None,
            [strip_commands(l) for l in open(require_file).readlines()])
      )
```
long_description直接保存在Readme文件中的内容，并加入到setup信息中。

上面定义两个基本的setup需要的函数，用于requirments依赖管理需要，会将当前库所依赖的所有第三方库同时添加到setup信息中。

开发环境中我们需要安装nose来进行程序的测试，所以setup定义中我们加入了test_suite和tests_require两个参数来进行测试执行使用，执行测试直接使用
**python setup.py test**即可，

```python
setup(name='animal',
      version='0.0.1',
      description='The simplest animal lib',
      long_description=long_description,
      long_description_content_type="text/markdown",
      url='http://github.com/zhangmingkai4315/animal',
      author='Mike Zhang',
      author_email='zhangmingkai.1989@gmail.com',
      license='MIT',
      packages=['animal'],
      zip_safe=False,

      test_suite="nose.collector",
      tests_require=['nose'],
      
      install_requires=req('requirments.txt'),

      classifiers=[
        "Programming Language :: Python :: 2.7",
        "Programming Language :: Python :: 3.0",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],

)
```
MANIFEST.in文件保存了需要同时一起打包的文件，比如说明和授权文件，以及python源码文件。

```
include README.md
include LICENSE
recursive-include *.txt *.py
```

#### 2. 测试程序包

测试文件写了两个简单的例子,执行nose的时候会自动的读取该文件，并执行。可以用来进行持续集成和部署的时候使用。

```python
from unittest import TestCase 
import animal

class TestAnimal(TestCase):
  def test_animal_init(self):
    bird = animal.Animal("bird")
    self.assertTrue(isinstance(bird,animal.Animal))
  
  def test_animal_validate(self):
    bird = animal.Animal("bird",legs=10)
    self.assertEqual(bird.legs_number(),10)
    with self.assertRaises(animal.ValidationError):
      bird = animal.Animal("bird",legs=100)
```

#### 3. 文件结构

整个包的基本结构如下所示：

```shell
.
├── LICENSE
├── MANIFEST.in
├── README.md
├── animal
│   ├── __init__.py
│   ├── animal.py
│   └── tests
│       └── test_animal.py
├── animal.egg-info
│   ├── ...
├── build
│   ├── ...
├── dist
│   ├── animal-0.0.1-py3-none-any.whl
│   └── animal-0.0.1.tar.gz
├── requirments.txt
├── setup.py
└── wheelhouse
    └── validator.py-1.2.5-py2.py3-none-any.whl

```

如果提交代码到git，记得将gitignore文件加入到版本库中：

```
# Compiled python modules.
*.pyc
# Setuptools distribution folder.
/dist/
# Python egg metadata, regenerated from source files by setuptools.
/*.egg-info
```
执行```pip install -e . ```可以将该包加入到当前python库中(类似于软链接，因此任何时候修改该包，不需要执行install安装)，直接被其他的程序文件所引用,这个可以通过在EPEL环境中直接```import animal```即可测试. 

#### 4.打包程序

安装必要的打包工具**setuptools**以及**wheel**
```
pip install setuptools wheel
```
执行下面的命令生成压缩文件和二进制wheel格式的包。可以将该包传递到任何位置直接安装使用。

```
python setup.py sdist bdist_wheel
```
上面生成的tar.gz包属于源代码包，而.whl则属于编译好的包，尽量同时提供上述的两个部分，特别是针对一些跨平台不同的应用包的管理。上述操作不会打包所有的依赖文件，仅包含当前的程序库本身。如果需要打包整个依赖用于离线安装则可以使用下面的操作完成：
```
mkdir wheelhouse
pip download -r requirments.txt -d wheelhouse

# 压缩并上传到待安装库的服务器上，执行安装程序
pip install animal-0.0.1-py3-none-any.whl --no-index --find
-links wheelhouse
```

注意wheelhouse仅仅用于离线安装，无需上传到PyPI上

#### 5.上传到PyPI

打包完成后，生成的tar包和whl包我们可以使用twine完成最后的上传管理，使用pip来安装twine程序。

```
pip install --user --upgrade twine
```

下载完成后即可直接使用twine来上传文件，这里如果没有注册过需要先完成注册过程。可以通过python setup.py register注册的话，也可以在线注册。上传我们之前生成文件的命令如下：


```
twine upload dist/* 
```

#### 6.生成二进制包

假如需要将程序打包成为二进制，用于在不同平台上执行，且考虑到以下的几种情况：
1. python环境可能在目标机器不存在
2. python使用的是修改过的CPython
3. 图形界面程序或者游戏等

二进制程序无需依赖目标机器上的python环境就可以执行。因此在使用上比较方便和管理。但是当前生成二进制的方式都不够完美，比如pyinstaller, pyexe, pyapp等，我发实现跨平台编译。因此需要借助于docker等方式生成编译环境后在容器中实现多个平台的编译执行。




