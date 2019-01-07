---
layout: post
title: Python多线程中的错误处理和限速器
date: 2018-12-24T11:32:41+00:00
author: mingkai
permalink: /2018/12/python-threads-with-error-and-throttle
categories:
  - python
---

python语言的主要实现版本CPython在实现线程的时候，加入了GIL(全局解释锁)，使得所有访问Python对象的线程都被一个全局的锁串行化， 强制任何时候只有一个解释锁可以运行。尽管多年来很多开发人员呼吁删除GIL，但是至少今天来说，该解释锁仍然存在于CPython的实现中，但像是IronPython和Jython中不包含这个问题（生产环境中使用较少）。

GIL的存在并没有完全的限制多线程的执行，GIL在一些阻塞操作的时候会被释放，以及在不使用Python/C API函数的一些C扩展的代码时候被释放，因此在比如网络操作等情况下多线程仍旧可以实现高效的运行。

使用多线程的主要场景包含：

- **构建用户响应式界面**： 多线程可以将一些复杂的工作推到后台执行，防止前端用户的响应被阻塞
- **多用户应用**：比如一些web服务器，同时需要处理多个用户的请求。主线程接收到用户的输入后，将启用一个新的线程来处理该用户的请求，主线程继续进入等待状态。
- **委派工作**： 将任务分发给多个线程执行，等待多个响应的查询结果，比如网络爬虫以及一些并发的API调用。

多线程的使用比多进程更加轻量级，而且多进程需要加载新的解释器消耗更多的资源。为了充分的利用多核心的优势，可以利用多进程和多线程混合的方式实现。

接下来使用多个实例来解释Python中并发编程的具体实现，我们将利用公共的WeatherAPI来进行天气的查询，查询

### 1. 单线程实现

首先为了查询OpenWeatherAPI的数据我们需要注册一个免费的账户，[点击注册链接](https://openweathermap.org) , 注册完成后将得到一个API token，用于接下来的查询。

```python
import requests
import time
OPEN_WEATHER_URL = "https://api.openweathermap.org/data/2.5/weather?q={}&appid="
TOKEN = "****"

def getWeatherData(city):
    query_url= OPEN_WEATHER_URL.format(city)+TOKEN
    r = requests.get(query_url)
    if r.status_code == 200:
        return {"name":city,"data":r.json()}
    return {"name":city,"error": r.status_code}
def show_result(results):
    for r in results:
        if "error" not in r:
            try:
                print("{} : {}".format(r["name"],r["data"]["weather"][0]["main"]))
            except:
                print("{} : no response".format(r["name"]))
        else:
            print("{} : no response".format(r["name"]))
def main():
    started_time = time.time()
    cities = ["Beijing","Shanghai","Tokyo","Amsterdam","London","Kenya"]
    results = []
    for city in cities:
        results.append(getWeatherData(city))
    show_result(results)
    print("Time elapsed: {:2f}s".format(time.time()-started_time))
if __name__ == "__main__":
    main()
```

程序输出结果显示，为了查询六个城市的天气情况，我们总计花费了大约11秒的时间

```json
Beijing : Clear
Shanghai : Clear
Tokyo : Clouds
Amsterdam : Clouds
London : Mist
Kenya : Rain
Time elapsed: 11.047014s
```

### 2. 多线程版本

网络查询其实是一个阻塞操作，直到收到上一次的查询后才能开启一个新的查询，为了加快程序的执行，我们需要利用多线程的方式加快程序的执行（GIL在网络阻塞的时候自动释放，用于执行新的线程）

对于多线程执行需要注意以下三点:

- 设置最大的线程执行数量
- 线程间通信
- 共享对象的多线程访问

```python
import requests
import time
from queue import Queue, Empty
from threading import Thread
 
OPEN_WEATHER_URL = "https://api.openweathermap.org/data/2.5/weather?q={}&appid="
TOKEN = "****"
Thread_Pool_Size = 6
def getWeatherData(city):
    query_url= OPEN_WEATHER_URL.format(city)+TOKEN
    r = requests.get(query_url)
    if r.status_code == 200:
        return {"name":city,"data":r.json()}
    return {"name":city,"error": r.status_code}


def worker(work_queue,result_queue):
    while not work_queue.empty():
        try:
            item = work_queue.get(block=False)
        except Empty:
            break
        else:
            result_queue.put(getWeatherData(item))
            work_queue.task_done()
def show_single_result(r):
    if "error" not in r:
        try:
            print("{} : {}".format(r["name"],r["data"]["weather"][0]["main"]))
        except:
            print("{} : no response".format(r["name"]))
    else:
        print("{} : no response".format(r["name"]))    

def main():
    threads = []
    work_queue = Queue()
    result_queue = Queue()
    started_time = time.time()
    cities = ["Beijing","Shanghai","Tokyo","Amsterdam","London","Kenya"]
    for city in cities: 
        work_queue.put(city)
    threads = [Thread(target=worker, args=(work_queue,result_queue)) for _ in range(Thread_Pool_Size)]
    for thread in threads:
        thread.start()
    work_queue.join()
    while not result_queue.empty():
        show_single_result(result_queue.get())
    print("Time elapsed: {:2f}s".format(time.time()-started_time))

if __name__ == "__main__":
    main()
```

在上面的代码执行中，总计的执行时间约降低了6倍左右，基本上等同于一次往返查询的时间，为了使得程序能够合理的使用资源，利用一个work_queue来管理资源的队列，并启用了线程池的方式创建6个工作线程来从队列中获取资源。

注意其中join和task_done的使用, 线程的join用来阻塞当前主线程的执行，防止主线程推出，Queue的join用来判断是否队列为空，如果不为空则阻塞在这里，每添加一个元素将使得计数增加1个，使用task_done来对计数减1，计数为0则恢复主线程执行。

### 3  错误捕获

当使用多线程的时候需要处理任何工作线程可能导致的错误异常，因为这些异常的出现可能导致工作线程退出，而不会被处理的数据将阻塞一些程序继续执行，比如上面的work中如果异常退出导致不能执行task_done则主线程中的join将会一直被阻塞。

修改上述程序的work进程，使得程序可以顺利的处理任何的异常的情况而不至于退出线程，最后我们打印的结果的时候可以进行类型判断，进而处理异常的错误

```python
def worker(work_queue,result_queue):
    while True:
        try:
            item = work_queue.get(block=False)
        except Empty:
            break
        else:
            try:
                result = getWeatherData(item)
            except Exception as e:
                result_queue.put(e)
            else:
                result_queue.put(result)
            finally:    
                work_queue.task_done()
                
....main()

    while not result_queue.empty():
        result = result_queue.get()
        if not isinstance(result, Exception):
            show_single_result(result)
        else:
            raise result
```



### 4. 速度限制

当我们将查询的数据扩大规模，并设置提高最大的线程数量的时候，有可能会发现结果,代表查询的速度太快，服务器拒绝了新的连接请求。

```python
Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/urllib3/connection.py", line 141, in _new_conn
    (self.host, self.port), self.timeout, **extra_kw)
  File "/usr/lib/python3/dist-packages/urllib3/util/connection.py", line 83, in create_connection
    raise err
  File "/usr/lib/python3/dist-packages/urllib3/util/connection.py", line 73, in create_connection
    sock.connect(sa)
BlockingIOError: [Errno 11] Resource temporarily unavailable

During handling of the above exception, another exception occurred:
```

使用一个全局的限速器可以实现上面的功能，每次我们想要处理新的任务的时候，需要先去该对象进行资源申请，申请资源的速度如果超出了设定的门限值则会暂停任务执行。

```python
class Throttle:
    def __init__(self, rate):
        self._lock = Lock()
        self.rate = rate
        self.tokens = 0
        self.last = 0
    def consume(self, amount=1):
        with self._lock:
            now = time.time()
            if self.last == 0:
                self.last = now
            elapsed = now - self.last
            if int(elapsed * self.rate):
                self.tokens += int(elapsed*self.rate)
                self.last = now
            if self.tokens > self.rate:
                self.tokens = self.rate
            if self.tokens >= amount:
                self.tokens -= amount
            else:
                amount = 0
            return amount
```

上述的限速器在使用的时候需要传递给每一个work，要求每一个work在执行任务前去申请资源获得token,如果无法获取，则循环等待。程序的完整代码如下：

```python
import requests
import time
from queue import Queue, Empty
from threading import Thread, Lock
 
OPEN_WEATHER_URL = "https://api.openweathermap.org/data/2.5/weather?q={}&appid="
TOKEN = "*****************"
Thread_Pool_Size = 100

class Throttle:
    def __init__(self, rate):
        self._lock = Lock()
        self.rate = rate
        self.tokens = 0
        self.last = 0
    def consume(self, amount=1):
        with self._lock:
            now = time.time()
            if self.last == 0:
                self.last = now
            elapsed = now - self.last
            if int(elapsed * self.rate):
                self.tokens += int(elapsed*self.rate)
                self.last = now
            if self.tokens > self.rate:
                self.tokens = self.rate
            if self.tokens >= amount:
                self.tokens -= amount
            else:
                amount = 0
            return amount


def getWeatherData(city):
    query_url= OPEN_WEATHER_URL.format(city)+TOKEN
    r = requests.get(query_url)
    if r.status_code == 200:
        return {"name":city,"data":r.json()}
    return {"name":city,"error": r.status_code}


def worker(work_queue,result_queue,throttle):
    while True:
        try:
            item = work_queue.get(block=False)
        except Empty:
            break
        else:
            try:
                while not throttle.consume():
                    pass
                result = getWeatherData(item)
            except Exception as e:
                result_queue.put(e)
            else:
                result_queue.put(result)
            finally:    
                work_queue.task_done()
def show_single_result(r):
    if "error" not in r:
        try:
            print("{} : {}".format(r["name"],r["data"]["weather"][0]["main"]))
        except:
            print("{} : no response".format(r["name"]))
    else:
        print("{} : no response".format(r["name"]))    

def main():
    threads = []
    work_queue = Queue()
    result_queue = Queue()
    started_time = time.time()
    throttle = Throttle(10)
    cities = ["Beijing","Shanghai","Tokyo","Amsterdam","London","Kenya"]*10
    for city in cities: 
        work_queue.put(city)
    threads = [Thread(target=worker, args=(work_queue,result_queue,throttle)) for _ in range(Thread_Pool_Size)]
    for thread in threads:
        thread.start()
    work_queue.join()
    # while threads:
    #     threads.pop().join()
    while not result_queue.empty():
        result = result_queue.get()
        if not isinstance(result, Exception):
            show_single_result(result)
        else:
            raise result
    print("Time elapsed: {:2f}s".format(time.time()-started_time))

if __name__ == "__main__":
    main()
```

通过测试发现当限速器设置10QPS的时候，100个请求的响应时间基本上约为10-11s左右，与预期的查询速度基本一致。


