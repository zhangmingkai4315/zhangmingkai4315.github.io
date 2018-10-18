---
id: 145
title: Django开发记录
date: 2018-09-05T20:59:10+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=145
permalink: /2018/09/django-development-knowledge-1/
categories:
  - python
tags:
  - django
  - python
  - web
format: aside
---
#### 1. Django代码风格

  1. 代码可读 
      1. 避免缩写变量名称
      2. 函数参数名称要有意义
      3. 写代码文档
      4. 添加部分注释
      5. 减少重复代码
      6. 函数和方法要尽量短小
  2. 参考PEP8规范，使用Flake8来检查一些错误（安装代码检查插件）
  3. 79个字符的限制，尽量不要超越该范围，特别是开源项目（Django文档中标记提示允许最大不超过119个字符长度的语句）
  4. 导入顺序（顺序应该按照标准库，第三方库和本地库顺序）可以借助于isort工具进行处理

    $ pip install isort
    $ isort -rc .
    

  1. 导入内容（尽量不要写&#42;导入所有，除非导入太多的内容）
  2. 相对导入方式，方便模块的迁移复用和重命名,如下面所示

<pre><code class="python">from django.views.generic import CreateView
# Relative imports of the 'cones' package
from .models import WaffleCone 
from .forms import WaffleConeForm 
from core.views import FoodMixin

class WaffleConeCreateView(FoodMixin, CreateView): 
    model = WaffleCone
    form_class = WaffleConeForm
</code></pre>

  1. 命名冲突问题

<pre><code class="python">from django.db.models import CharField as ModelCharField from django.forms import CharField as FormCharField
</code></pre>

  1. 四个空格代表缩进，变量名应该是小写并以&#95;连接
  
    更多内容：https://docs.djangoproject.com/en/1.11/internals/contributing/writing-code/coding-style/

  2. 使用eslint来检查javascript的代码风格

  3. 使用stylelint来价差css的格式

#### 2. 环境配置

尽量在开发和生产线上使用同样的数据库引擎，因为不同的引擎对于字段和域的类型和限制不同，可能会执行过程中出现问题。

> 不要在生产环境使用SQLite,开发环境可以使用docker启动一个mysql或者postgres!

    docker run --name=django-mysql -d -p 3306:3306 -e "MYSQL_ROOT_PASSWORD=123456" mysql:5.7
    

使用pip和virtualenv来管理版本冲突，每个项目独立自己的运行环境和依赖库，pip在python3.4或者更高版本上已经内置了，所以不需要独立安装。同时可以安装**virtualenvwrapper**， 简化环境变量使用,但不是必须的。

通过pip来安装django和其他的依赖

    virtualenv venv
    source venv/bin/active
    pip install Django==1.11
    pip freeze > requirements.txt
    

将目录保存为如下的结构：

    .
    ├── Makefile
    ├── README.md
    ├── config
    │   ├── __init__.py
    │   ├── settings
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── docs
    ├── icecreamratings
    │   ├── media
    │   ├── productions
    │   ├── profiles
    │   ├── ratings
    │   ├── static
    │   └── templates
    ├── manage.py
    ├── requirements.txt
    

其中icecreamratins是项目目录，包含了一些文件夹，
  
&#8211; media用来存储所有的图片或者视频等等，当项目较大的时候，应该将该目录单独的转移到一个独立的静态文件服务器上，通过API来读取和写入。
  
&#8211; productions用于产品的管理，
  
&#8211; profile用户的管理，
  
&#8211; ratings用于用户的评分，
  
&#8211; static用于项目中html css和js等，项目较大的时候也是转移到指定的服务器
  
&#8211; template存放项目的模板。

#### 3.Django设计

Django项目和Django app的区别在于app是一个小的库来完成项目中的某一个方面，一个项目一般都会包含很多个app，INSTALLED_APPS是指的是一系列启用的Django Apps。

> 一个优秀的Django app，应该只做一件事情，并且能够做好。

设计一个甜品店的官方网站，app可能分别用于不同的目的，比如：

  * blog app用于官方的blog内容
  * events app用于发布一些告示或者提示信息
  * shop app 用于允许在线购买甜品
  * tickets app用于活动票卷的管理
  * &#8230;

命名app应该按照功能命名，或者依据app的主要数据模型进行命名，或者根据url来进行命名管理。 一个典型的app模块应该包含以下几个部分：

    blogs/
       ├── __init__.py
       ├── admin.py
       ├── forms.py
       ├── management/
       ├── migrations/
       ├── models.py
       ├── templatetags/
       ├── tests/
       ├── urls.py
       ├── views.py
    

#### 4. Django设置

对于settings应该考虑遵循如下的规则：

  * 所有的设置项目应该使用版本控制，用于追踪任何更改
  * 不要重复，尽量从基类配置中继承配置
  * 保持机密的信息安全

SECRET_KEY要保持安全，设置一个全新的独一无二的值，并且应该设置到版本控制之外保存，防止泄露。

对于设置可以按照下面的方式进行保存，一个settings文件夹下面包含所有环境的配置，另外除了这些基本的配置，还可以加入比如ci.py来进行持续集成等的配置内容

    settings/
       ├── __init__.py
       ├── base.py
       ├── local.py
       ├── staging.py
       ├── test.py
       ├── production.py
    

启动python使用本地环境配置可以用下面的语句

    python manage.py runserver --settings=apps.settings.local
    

这样在不同环境中管理不同的配置仅需要修改命令即可，不需要修改任何的代码项。另外假如不同的人开发环境不同，可以设置多个local文件，比如local_mike.py 但是都应该遵循最小复制原则，尽量从原始的local中继承修改。

机密的配置信息应该通过环境变量进行保存，而不是加入到版本系统中。 利用一段脚本将所需要的信息导入到系统中

<pre><code class="shell">  export SOME_SECRET_KEY=1c3-cr3am-15-yummy
  export AUDREY_FREEZER_KEY=y34h-r1ght-d0nt-t0uch-my-1c3-cr34m
  # unset ...
</code></pre>

在web项目中永远不要设置绝对路径应该使用相对路径比如

<pre><code class="python"># Build paths inside the project like this: os.path.join(BASE_DIR, ...)
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
ROOT_DIR = os.path.dirname(BASE_DIR)
MEDIA_DIR = os.path.join(ROOT_DIR,PROJECT_NAME,'media')
STATIC_DIR = os.path.join(ROOT_DIR,PROJECT_NAME,'static')
</code></pre>

#### 5.Django Model

模型设计要考虑全局，不要草率的创建一些模型，后期维护和修改的成本都会很大。可以考虑使用一些扩展库来创建模型比如`django-model-utils`, 每一个app所包含的模型应该保持在5个以内，超过该数量的app应该考虑分离app到更小的app上去。对于很多模型都需要的字段，可以独立出来抽象模型类。

比如下面的TimeStampedModel类：

<pre><code class="python">class TimeStampedModel(models.Model):
  """
  An abstract base class model that provides 
  self updating created and modified fields
  """

  created = models.DateTimeField(auto_now_add=True)
  modified = models.DateTimeField(auto_now=True)

  class Meta:
    abstract = True
</code></pre>

其他的models都可以继承自该模型并获得这个字段，这种方式最简单直接，另外的方式比如代理模式（重命名一个模型，并设置新的属性）和多表继承（建立1对1的父子联系，可能造成性能问题）都不够简洁和直接。

使用django_admin创建一个app后，修改models然后执行makemigrations 当生成完成迁移文件后运行migrate来执行数据库迁移

> 一定要做好数据备份工作，在stage环境下执行测试，确保测试的数据和实际数据量相似，对于使用MySQL，schema改变无法实现事务的支持，所以无法执行rollbacks。 执行改变时候，数据库可以设置为只读模式，线上进行迁移的时间往往会很长。

数据库设计的几个原则：

  1. 数据规范化，使用模型之间的关系去除重复数据
  2. 尽量不要使用去规范化方式存储数据，导致数据可能重复和丢失
  3. 去规范化的数据可以用于构建缓存数据，或者减少数据库压力

* * *

数据库设计中Model参数选择问题， null=True和blank=True的区别?

  * null=True 允许在数据库中的所属字段可以设置 NULL (versus NOT NULL). 比如DateTimeField 或者 ForeignKey可以设置为NULL.

  * blank=True主要用于限制表单的行为，是否允许为空不去设置.

  * 两个结合使用的时候，要注意如果允许表单为空，则一般在数据库层面也要设置为允许为Null, 但是比如 CharFields 和 TextFields,实际存储的是空字符串 (&#8221;).

* * *

  1. 使用BinaryField的时候要考虑性能问题，不要存放太多的数据到该字段中，如果太大的话 考虑使用文件的形式，并保存文件的链接到该字段。
    
    **Fat Models** ：特指的是那些将业务逻辑引入Model的方式，使model不仅包含数据库属性定义，还包含了一些基本的操作和统计或者业务逻辑等等。即有优点也有缺点，优点是可以在多个view或者模板中被使用，但是缺点也是导致model过于庞大，测试和维护的成本都比较大。可以考虑使用Mixins来提升抽象，减少Model的复杂性

> 参考链接 http://blog.kevinastone.com/django-model-behaviors.html

#### 查询和数据层

仅仅在View层使用get\_object\_or_404()函数其他地方禁止使用。并且不需要使用异常捕获，因为内部已经做了处理，其他地方尽量使用try&#8230;catch&#8230;来捕获相关的异常内容

使用过滤器，利用django内置的一些功能去进行数据的查询比如：

<pre><code class="python">from django.db.models import F
from models.customers import Customer

customs = Customer.objects.filter(scoops_ordered__gt=F('store_visted'))
</code></pre>

等价的SQL表达式： `SELECT * from customers_customer where scoops_ordered > store_visits`.

使用raw()函数来写SQL语句时候，牢记大部分的SQL查询都可以使用ORM来实现，除非SQL本身的一些高级用法，或者使用SQL比直接调用大量的ORM查询更简单有效的时候。

> 更多的函数表达式：https://docs.djangoproject.com/en/1.11/ref/models/database-functions/

##### 索引

索引的使用db_index=True, 尽管简单但是可以在开始时候先不去创建，等需要的时候再给字段增加索引。当考虑增加索引时候，以下为一些建议：

  * 频繁查询的字段可以考虑增加索引
  * 处于查询where的位置或者join on 的位置的字段
  * 单表索引尽量不要超过5个索引
  * 任何索引都会使得查询更快，但是也会导致插入更慢
  * 对于字段值很少的，比如状态或者boolean型的字段，增加索引弊大于利

##### 事务操作

django支持事务操作，在http查询层面配置数据库如下所示：

    DATABASES = {
           'default': {
               ...
               'ATOMIC_REQUESTS': True,
           },
    }
    
    

所有的http请求将被包含在单一的事务中，但是会导致查询读取数据库也会比较慢，这就是事务的代价。对于写多的数据库操作，会有一定的帮助，防止出现数据的不完整。对于事务的操作，可以按照粒度进行分布执行例如：

<pre><code class="python">from django.db import transaction
from django.http import HttpResponse
from django.shortcuts import get_object_or_404 
from django.utils import timezone

from .models import Flavor

@transaction.non_atomic_requests
def posting_flavor_status(request, pk, status): 
    flavor = get_object_or_404(Flavor, pk=pk)
    # This will execute in autocommit mode (Django's default).
    flavor.latest_status_change_attempt = timezone.now()
    flavor.save()
    with transaction.atomic():
    # This code executes inside a transaction. 
        flavor.status = status    
        flavor.latest_status_change_success = timezone.now() 
        flavor.save()
        return HttpResponse('Hooray')
       # If the transaction fails, return the appropriate status
    return HttpResponse('Sadness', status_code=400)
</code></pre>

仅仅对于需要的操作比如更新数据进行事务处理、

如果使用MySQL，你的表可能不支持事务，这依赖于使用的存储引擎类型（InnoDB或者MyISAM），如果不支持的话，所有事务代码将自动使用autocommit模式进行处理。