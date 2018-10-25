---
layout: post
title: 如何编写Prometheus Exporter
date: 2018-07-21T11:32:41+00:00
author: mingkai
permalink: /2018/07/how-to-implement-a-prometheus-exporter/
categories:
  - devops
tags:
  - devops
---

Exporter 是基于 Prometheus 实施的监控系统中重要的组成部分，承担数据指标的采集工作，官方的 exporter 列表中已经包含了常见的绝大多数的系统指标监控，比如用于机器性能监控的 node_exporter, 用于网络设备监控的 snmp_exporter 等等。这些已有的 exporter 对于监控来说，仅仅需要很少的配置工作就能提供完善的数据指标采集。

有时我们需要自己去写一些与业务逻辑比较相关的指标监控，这些指标无法通过常见的 exporter 获取到。比如我们需要提供对于 DNS 解析情况的整体监控，了解如何编写 exporter 对于业务监控很重要，也是完善监控系统需要经历的一个阶段。接下来我们就介绍如何编写 exporter, 本篇内容编写的语言为 golang, 官方也提供了 python, java 等其他的语言实现的库，采集方式其实大同小异。

#### 搭建环境

首先确保机器上安装了 go 语言(1.7 版本以上)，并设置好了对应的 GOPATH。接下来我们就可以开始编写代码了。以下是一个简单的 exporter

下载对应的 prometheus 包

```
go get github.com/prometheus/client_golang/prometheus/promhttp
```

程序主函数:

```go
package main

import (
	"log"
	"net/http"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)
func main() {
	http.Handle("/metrics", promhttp.Handler())
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

这个代码中我们仅仅通过 http 模块指定了一个路径，并将 client_golang 库中的 promhttp.Handler()作为处理函数传递进去后，就可以获取指标信息了,两行代码实现了一个 exporter。这里内部其实是使用了一个默认的收集器将通过 NewGoCollector 采集当前 Go 运行时的相关信息比如 go 堆栈使用,goroutine 的数据等等。 通过访问http://localhost:8080/metrics即可查看详细的指标参数。

上面的代码仅仅展示了一个默认的采集器，并且通过接口调用隐藏了太多实施细节，对于下一步开发并没什么作用，为了实现自定义的监控我们需要先了解一些基本概念。

#### 指标类别

Prometheus 中主要使用的四类指标类型，如下所示

- Counter (累加指标)
- Gauge (测量指标)
- Summary (概略图)
- Histogram (直方图)

Counter 一个累加指标数据，这个值随着时间只会逐渐的增加，比如程序完成的总任务数量，运行错误发生的总次数。常见的还有交换机中 snmp 采集的数据流量也属于该类型，代表了持续增加的数据包或者传输字节累加值。

Gauge 代表了采集的一个单一数据，这个数据可以增加也可以减少，比如 CPU 使用情况，内存使用量，硬盘当前的空间容量等等

Histogram 和 Summary 使用的频率较少，两种都是基于采样的方式。另外有一些库对于这两个指标的使用和支持程度不同，有些仅仅实现了部分功能。这两个类型对于某一些业务需求可能比较常见，比如查询单位时间内：总的响应时间低于 300ms 的占比，或者查询 95%用户查询的门限值对应的响应时间是多少。 使用 Histogram 和 Summary 指标的时候同时会产生多组数据，\_count 代表了采样的总数，\_sum 则代表采样值的和。 \_bucket 则代表了落入此范围的数据。

下面是使用 historam 来定义的一组指标，计算出了平均五分钟内的查询请求小于 0.3s 的请求占比总量的比例值。

```
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) by (job)
/
  sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

如果需要聚合数据，可以使用 histogram. 并且如果对于分布范围有明确的值的情况下（比如 300ms），也可以使用 histogram。但是如果仅仅是一个百分比的值（比如上面的 95%），则使用 Summary

#### 定义指标

这里我们需要引入另一个依赖库

```
go get github.com/prometheus/client_golang/prometheus
```

下面先来定义了两个指标数据，一个是 Guage 类型， 一个是 Counter 类型。分别代表了 CPU 温度和磁盘失败次数统计，使用上面的定义进行分类。

```
    cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "cpu_temperature_celsius",
		Help: "Current temperature of the CPU.",
	})
	hdFailures = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "hd_errors_total",
			Help: "Number of hard-disk errors.",
		},
		[]string{"device"},
	)
```

这里还可以注册其他的参数，比如上面的磁盘失败次数统计上，我们可以同时传递一个 device 设备名称进去，这样我们采集的时候就可以获得多个不同的指标。每个指标对应了一个设备的磁盘失败次数统计。

#### 注册指标

```
func init() {
	// Metrics have to be registered to be exposed:
	prometheus.MustRegister(cpuTemp)
	prometheus.MustRegister(hdFailures)
}
```

使用 prometheus.MustRegister 是将数据直接注册到 Default Registry，就像上面的运行的例子一样，这个 Default Registry 不需要额外的任何代码就可以将指标传递出去。注册后既可以在程序层面上去使用该指标了，这里我们使用之前定义的指标提供的 API（Set 和 With().Inc）去改变指标的数据内容

```
func main() {
	cpuTemp.Set(65.3)
	hdFailures.With(prometheus.Labels{"device":"/dev/sda"}).Inc()

	// The Handler function provides a default handler to expose metrics
	// via an HTTP server. "/metrics" is the usual endpoint for that.
	http.Handle("/metrics", promhttp.Handler())
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

其中 With 函数是传递到之前定义的 label="device"上的值，也就是生成指标类似于

```
cpu_temperature_celsius 65.3
hd_errors_total{"device"="/dev/sda"} 1
```

当然我们写在 main 函数中的方式是有问题的，这样这个指标仅仅改变了一次，不会随着我们下次采集数据的时候发生任何变化，我们希望的是每次执行采集的时候，程序都去自动的抓取指标并将数据通过 http 的方式传递给我们。

#### Counter 数据采集实例

下面是一个采集 Counter 类型数据的实例，这个例子中实现了一个自定义的，满足采集器(Collector)接口的结构体，并手动注册该结构体后，使其每次查询的时候自动执行采集任务。

我们先来看下采集器 Collector 接口的实现

```
type Collector interface {
    // 用于传递所有可能的指标的定义描述符
    // 可以在程序运行期间添加新的描述，收集新的指标信息
    // 重复的描述符将被忽略。两个不同的Collector不要设置相同的描述符
	Describe(chan<- *Desc)

    // Prometheus的注册器调用Collect执行实际的抓取参数的工作，
    // 并将收集的数据传递到Channel中返回
    // 收集的指标信息来自于Describe中传递，可以并发的执行抓取工作，但是必须要保证线程的安全。
	Collect(chan<- Metric)
}
```

了解了接口的实现后，我们就可以写自己的实现了，先定义结构体，这是一个集群的指标采集器，每个集群都有自己的 Zone,代表集群的名称。另外两个是保存的采集的指标。

```
type ClusterManager struct {
    Zone         string
    OOMCountDesc *prometheus.Desc
    RAMUsageDesc *prometheus.Desc
}
```

我们来实现一个采集工作,放到了**ReallyExpensiveAssessmentOfTheSystemState**函数中实现，每次执行的时候，返回一个按照主机名作为键采集到的数据，两个返回值分别代表了 OOM 错误计数，和 RAM 使用指标信息。

```
func (c *ClusterManager) ReallyExpensiveAssessmentOfTheSystemState() (
    oomCountByHost map[string]int, ramUsageByHost map[string]float64,
) {
	oomCountByHost = map[string]int{
		"foo.example.org": int(rand.Int31n(1000)),
		"bar.example.org": int(rand.Int31n(1000)),
	}
	ramUsageByHost = map[string]float64{
		"foo.example.org": rand.Float64() * 100,
		"bar.example.org": rand.Float64() * 100,
	}
	return
}
```

实现 Describe 接口，传递指标描述符到 channel

```
// Describe simply sends the two Descs in the struct to the channel.
func (c *ClusterManager) Describe(ch chan<- *prometheus.Desc) {
    ch <- c.OOMCountDesc
    ch <- c.RAMUsageDesc
}
```

Collect 函数将执行抓取函数并返回数据，返回的数据传递到 channel 中，并且传递的同时绑定原先的指标描述符。以及指标的类型（一个 Counter 和一个 Guage）

```
func (c *ClusterManager) Collect(ch chan<- prometheus.Metric) {
    oomCountByHost, ramUsageByHost := c.ReallyExpensiveAssessmentOfTheSystemState()
    for host, oomCount := range oomCountByHost {
        ch <- prometheus.MustNewConstMetric(
            c.OOMCountDesc,
            prometheus.CounterValue,
            float64(oomCount),
            host,
        )
    }
    for host, ramUsage := range ramUsageByHost {
        ch <- prometheus.MustNewConstMetric(
            c.RAMUsageDesc,
            prometheus.GaugeValue,
            ramUsage,
            host,
        )
    }
}
```

创建结构体及对应的指标信息,NewDesc 参数第一个为指标的名称，第二个为帮助信息，显示在指标的上面作为注释，第三个是定义的 label 名称数组，第四个是定义的 Labels

```
func NewClusterManager(zone string) *ClusterManager {
    return &ClusterManager{
        Zone: zone,
        OOMCountDesc: prometheus.NewDesc(
            "clustermanager_oom_crashes_total",
            "Number of OOM crashes.",
            []string{"host"},
            prometheus.Labels{"zone": zone},
        ),
        RAMUsageDesc: prometheus.NewDesc(
            "clustermanager_ram_usage_bytes",
            "RAM usage as reported to the cluster manager.",
            []string{"host"},
            prometheus.Labels{"zone": zone},
        ),
    }
}
```

执行主程序

```
func main() {
    workerDB := NewClusterManager("db")
    workerCA := NewClusterManager("ca")

    // Since we are dealing with custom Collector implementations, it might
    // be a good idea to try it out with a pedantic registry.
    reg := prometheus.NewPedanticRegistry()
    reg.MustRegister(workerDB)
    reg.MustRegister(workerCA)
}
```

如果直接执行上面的参数的话，不会获取任何的参数，因为程序将自动推出，我们并未定义 http 接口去暴露数据出来，因此数据在执行的时候还需要定义一个 httphandler 来处理 http 请求。

添加下面的代码到 main 函数后面，即可实现数据传递到 http 接口上：

```
	gatherers := prometheus.Gatherers{
		prometheus.DefaultGatherer,
		reg,
	}

	h := promhttp.HandlerFor(gatherers,
		promhttp.HandlerOpts{
			ErrorLog:      log.NewErrorLogger(),
			ErrorHandling: promhttp.ContinueOnError,
		})
	http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		h.ServeHTTP(w, r)
	})
	log.Infoln("Start server at :8080")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Errorf("Error occur when start server %v", err)
		os.Exit(1)
	}
```

其中 prometheus.Gatherers 用来定义一个采集数据的收集器集合，可以 merge 多个不同的采集数据到一个结果集合，这里我们传递了缺省的 DefaultGatherer，所以他在输出中也会包含 go 运行时指标信息。同时包含 reg 是我们之前生成的一个注册对象，用来自定义采集数据。

promhttp.HandlerFor()函数传递之前的 Gatherers 对象，并返回一个 httpHandler 对象，这个 httpHandler 对象可以调用其自身的 ServHTTP 函数来接手 http 请求，并返回响应。其中 promhttp.HandlerOpts 定义了采集过程中如果发生错误时，继续采集其他的数据。

尝试刷新几次浏览器获取最新的指标信息

```
clustermanager_oom_crashes_total{host="bar.example.org",zone="ca"} 364
clustermanager_oom_crashes_total{host="bar.example.org",zone="db"} 90
clustermanager_oom_crashes_total{host="foo.example.org",zone="ca"} 844
clustermanager_oom_crashes_total{host="foo.example.org",zone="db"} 801
# HELP clustermanager_ram_usage_bytes RAM usage as reported to the cluster manager.
# TYPE clustermanager_ram_usage_bytes gauge
clustermanager_ram_usage_bytes{host="bar.example.org",zone="ca"} 10.738111282075208
clustermanager_ram_usage_bytes{host="bar.example.org",zone="db"} 19.003276633920805
clustermanager_ram_usage_bytes{host="foo.example.org",zone="ca"} 79.72085409108028
clustermanager_ram_usage_bytes{host="foo.example.org",zone="db"} 13.041384617379178
```

每次刷新的时候，我们都会获得不同的数据，类似于实现了一个数值不断改变的采集器。当然，具体的指标和采集函数还需要按照需求进行修改，满足实际的业务需求。
