---
id: 128
title: ElasticSearch集群管理
date: 2018-08-13T09:41:10+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=128
permalink: /2018/08/elasticsearch-cluster-management/
categories:
  - devops
tags:
  - elasticseach
  - 工具
  - 运维
format: aside
---
本文主要介绍ElasticSearch在分布式环境中的使用,包含如果实现集群管理,分片以及水平扩展,如何应对硬件失效.

ElasticSearch本身原生支持分布式,内置了多节点的管理和高可用机制. 应用开发不需要关系内部数据的分布式存储.在ElasticSearch**node**代表了一个运行了ElasticSearch的实例的机器,cluster则代表了拥有相同**cluster.name**的一个或者多个节点组成.

正常工作时候会选举其中一个node作为master管理集群的数据变更(index管理),节点管理, 但是不需要参与文档级别内的数据管理.

> GET /_cluster/health

    {
      "cluster_name": "docker-cluster",
      "status": "yellow",
      "timed_out": false,
      "number_of_nodes": 1,
      "number_of_data_nodes": 1,
      "active_primary_shards": 5,
      "active_shards": 5,
      "relocating_shards": 0,
      "initializing_shards": 0,
      "unassigned_shards": 5,
      "delayed_unassigned_shards": 0,
      "number_of_pending_tasks": 0,
      "number_of_in_flight_fetch": 0,
      "task_max_waiting_in_queue_millis": 0,
      "active_shards_percent_as_number": 50
    }
    

其中状态含义如下:
  
&#8211; green : 所有主和复制分片均ok
  
&#8211; yellow : 所有主分片运行正常,复制分片状态未激活
  
&#8211; red : 主和复制分片均存在问题

##### Shards

index 指的是存储相关数据的逻辑命名空间, 而实际指向了物理存储单元shards中(所有数据的分片存储单元),文档实际存储和检索在shards中,但是我们使用的话直接面先的是index级别.shards实际是一个lucene实例.ElasticSearch自动分配shards在所有节点中存储.

shard可以是主shard和复制shard. 每一个文档都属于一个主shard,复制shard保存一份主shard的复制,除了提供冗余外,还可以直接用于读取和搜索使用, 复制shards的数量可以任意变化.而主shard的数量则比较固定.

创建一个index, 默认会将所有数据存储在5个主shard上,但是可以配置比如下面的命令创建

    PUT /blogs
    {
      "settings": {
        "number_of_replicas": 1,
        "number_of_shards": 3
      }
    }
    

当有任何的新的node加入到集群中的时候,自动调整replicas的分配,当存在多个节点的时候,其中一个被作为master, 同时数据的主shard和复制shard将自动调整分配位置

![node的自动调整](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0204.png)

我们可以通过修改配置新的复制的数量

    PUT /blogs/_settings
    {
       "number_of_replicas" : 2
    }
    

![enter image description here](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0205.png)

配置更多的复制集合,可以允许我们容忍更多节点的失效.但是如果想要更快的处理效率,还是需要增加跟多的硬件资源.

##### 失效

当单一节点失效的时候,假如主节点,则新的主节点将被选举出来,同时数据主shard的失效,将通过复制shard来补充(转变状态)

![enter image description here](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0206.png)

但是此时的状态仍不会变为green,因为我们要求的三份复制集合,现在只有两份了.所以不是所有的复制集合都是可用的.

当node1重新启动的时候,数据将会重新被复制,并且复制shard使得满足条件,集群状态重新变为green.

#### 文档存储

存储
  
正常情况下我们存储一个文档对象,可以声明他的ID值(如果已经存在,则更新)

    POST /website/blog/123
    {
      "title": "My first blog entry",
      "text":  "Just trying this out...",
      "date":  "2014/01/01"
    }
    

但是当我们不去考虑ID的时候,ElasticSearch将会自动帮助生成一个20字符长度的URL安全的GUID(多节点同时生成防止冲突)

    {
      "_index": "website",
      "_type": "blogs",
      "_id": "U8jyQWIBQOPNwDp6PeAA",
      "_version": 1,
      "result": "created",
      "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
      },
      "_seq_no": 0,
      "_primary_term": 1
    }
    

如果我们仅想在不存在该ID对象的时候执行插入操作,则可以使用使用带有标记的插入命令

> PUT /website/blog/123?op_type=create
    
> 或者
    
> PUT /website/blog/123/_create

如果项目已经存在则返回**409 Conflict**,以及错误消息,代表文档已经存在.

提取文档的时候我们可以使用**_source**来定义返回的内容字段包含,类似与mysql中`select title from blogs`

    GET /website/blogs/VMjyQWIBQOPNwDp6hOAR?_source=title
    

查看是否存在

    HEAD /website/blogs/VMjyQWIBQOPNwDp6hOAR
    

修改内容

    PUT /website/blog/123
    {
      "title": "My first blog entry",
      "text":  "I am starting to get the hang of this...",
      "date":  "2014/01/02"
    }
    

这里的PUT将会替换所有的文档内容,如果需要部分更新的话则使用POST 实现,另外还支持脚本修改,比如自增或者批处理,详见[官方文档](https://www.elastic.co/guide/en/elasticsearch/guide/current/partial-updates.html)

     POST /website/blogs/U8jyQWIBQOPNwDp6PeAA/_update
     {
       "doc" : {
          "tags" : [ "testing" ],
          "views": 0
       }
     }
    

如果删除文档的话则直接使用如下的语法即可,如果内容不存在,则返回404代码.

> DELETE /website/blog/123

##### 处理冲突与版本

当有多个客户端连接elasticsearch时候, 由于数据存储和检索位置不同,可能导致冲突的发生如下图所示

![enter image description here](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0301.png)

对于上述问题的处理方案,有两种:

  * 乐观型并发控制: 假设这种情况发生概率较低,所以不会阻塞任何人的操作,如果数据出现修改(在一次读和写之间),由应用决定怎样去解决冲突,重新尝试或者刷新数据. 
  * 悲观型并发控制: 主要是用在关系型数据库, 提供锁来避免冲突的发生. 

ElasticSearch使用版本**_version字段**控制,来防止旧数据对于新的数据的覆盖(特别是多node的时候,数据传播到其他节点是无序的)

    PUT /website/blog/1?version=1 
    {
      "title": "My first blog entry",
      "text":  "Starting to get the hang of this..."
    }
    

仅当版本号为1的时候进行更新.

##### 外部版本

如果我们数据来自与其他地方,比如其他的数据库,如果数据库包含一个更新时间,我们可以使用这个更新时间作为外部版本,比如

    PUT /website/blog/2?version=5&version_type=external
    {
      "title": "My first external blog entry",
      "text":  "Starting to get the hang of this..."
    }
    

同时获取多个文档我们可以借助与_mget方法实现

    POST /_mget
    {
       "docs" : [
          {
             "_index" : "website",
             "_type" :  "blogs",
             "_id" :    1
          },
          {
             "_index" : "website",
             "_type" :  "pageviews",
             "_id" :    1,
             "_source": "views"
          }
       ]
    }
    

批量操作,如果需要同时执行多个操作的时候,可以使用_bulk来实现批量的操作

    POST /_bulk
    { "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
    { "create": { "_index": "website", "_type": "blog", "_id": "123" }}
    { "title":    "My first blog post" }
    { "index":  { "_index": "website", "_type": "blog" }}
    { "title":    "My second blog post" }
    { "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
    { "doc" : {"title" : "My updated blog post"} }