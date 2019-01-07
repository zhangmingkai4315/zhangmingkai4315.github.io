---
layout: post
title: Python并发处理
date: 2019-01-06T11:32:41+00:00
author: mingkai
permalink: /2019/01/python-concurrent-ways
categories:
  - python
---

当我们尝试进行读写文件或者建立网络socket进行数据读写的时候，一般需要花费毫秒级别左右的时间，在此期间程序被迫暂停，等待操作完成的信号，这段等待时间称之为**IO等待时间**。并发编程则允许我们在IO等待时间执行其他的操作，从而提高程序执行的效率。即便是在单核单线程 上也可以实现并发！



下面的图示展示了串行，并发和并行的区别，其中并发又包含并行，并行是一种特殊的并发，需要依赖于计算机本身的硬件系统支持，比如多核CPU，这样程序可以同时工作，但是并发也可以在单线程上执行，通过切分时间片的形式，在单核上实现上下文之间的不断切换实现”同时执行“。

![](https://1843914808.rsc.cdn77.org/wp-content/uploads/2016/11/GCD-1.png?x91151)



异步编程是实现并发的一种方式，通过内部实现一个事件循环的方式，实现多个任务的异步非阻塞执行，Nodejs本身就是构建在一个事件循环上面实现的，因此代码基本上都是以异步的方式进行处理。但是Python中实现却比较麻烦，对于Python2.7版本中需要借助于第三方库比如gevent或者tornado实现，python3.4上引入了asyncio库实现起来较简单一些。



### 1. gevent实现并发

gevent库通过对于标准的I/O函数做了MonkeyPatch（打补丁）的方式实现对于原I/O操作的修改，使得这些操作成为异步的操作，因此我们可以使用标准的IO就能获得异步的实现。

gevent内部通过greenlet来进行并发处理，greenlet是一种协程，所有的greenlet可以在一个物理线程上运行，也就是通过事件循环的方式在不同的greenlet中进行切换执行，下面的是一个采用gevent进行异步操作的实例, 其中synchronous为串行执行，因此可以看到返回的时间依次增加，而asynchronous则采用绑定gevent的方式执行（类似于JavaScript中的Promise的方式处理异步操作）

```python
import gevent.monkey
gevent.monkey.patch_socket()

import gevent
import urllib2
import json

def fetch(pid):
    response = urllib2.urlopen('http://worldtimeapi.org/api/timezone/Asia/Shanghai')
    result = response.read()
    json_result = json.loads(result)
    datetime = json_result['datetime']

    print('Process %s: %s' % (pid, datetime))
    return json_result['datetime']

def synchronous():
    for i in range(1,10):
        fetch(i)

def asynchronous():
    threads = []
    for i in range(1,10):
        threads.append(gevent.spawn(fetch, i))
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
```

同时为了保证性能和传输的稳定，我们有时候会对于并发进行限速控制，保证仅有固定数量的greenlet在运行，这里不再提供代码实例，可以查询gevent中原生自带的Semaphore信号量来保证，或者使用另一个第三方库grequest（imap函数）来协助完成。

### 2. tornodo实现并发

tornodo主要针对的是I/O密集型的应用，比如提供一个高性能并发的web服务器，使用tornado多是提供一个web服务器，对于异步的查询请求如下所示：

```python
import tornado.ioloop
from tornado.httpclient import AsyncHTTPClient

def handle_response(response):
    if response.error:
        print("Error:", response.error)
    else:
        data = response.body
        print data


http_client = AsyncHTTPClient()
url = "http://worldtimeapi.org/api/timezone/Asia/Shanghai"
for i  in range(1,10):
    http_client.fetch(url, handle_response)
    
tornado.ioloop.IOLoop.instance().start()
```

程序启动后，执行所有的异步操作，但是IOLoop本身启动还需要设置一定的机制去退出整个的循环，因此实际使用配置比较繁琐，不如gevent简单。

### 3. Python3与asyncio库的使用

Python3.4之后标准库引入了asyncio库用来原生支持并发的异步编程方式，使用async/await来实现代码的编写，在Python3版本的很多第三方库中被使用作为编写异步高性能网络和web服务器的底层代码。重新



```go
import asyncio
import aiohttp
import time 


async def fetch(session,url):
    async with session.get(url) as response:
        return await response.text()


async def fetch_data():
    url = 'http://worldtimeapi.org/api/timezone/Asia/Shanghai'
    async with aiohttp.ClientSession() as session:
        body = await fetch(session, url)
        print(body)



if __name__ == "__main__":
    start = time.time()
    loop = asyncio.get_event_loop()
    tasks = []
    for i in range(10):
        task = asyncio.ensure_future(fetch_data())
        tasks.append(task)
    loop.run_until_complete(asyncio.wait(tasks))

    end = time.time()
    print("time cost {}".format(end-start))
```



### 附录

1. http://worldtimeapi.org/
2. https://pawelmhm.github.io/asyncio/python/aiohttp/2016/04/22/asyncio-aiohttp.html
3. https://hackernoon.com/a-simple-introduction-to-pythons-asyncio-595d9c9ecf8c
4. https://github.com/tornadoweb/tornado/issues/1791