---
id: 55
title: Prometheus源码笔记-notifier模块
date: 2018-07-23T12:54:53+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=55
permalink: /2018/07/prometheus-notifier/
categories:
  - devops
tags:
  - golang
  - prometheus
  - readcode
  - 监控系统
---
Prometheus服务器根据规则发送告警到AlertManager，告警管理器根据接收到的告警进行分发处理，经过分组或者过滤等发送到邮件或者其他接收系统中。对于告警除了发送告警信息外，管理器还支持以下的几种操作：

  1. **分组处理**会将相似的告警进行聚合，比如一个数据库不可用的情况下，导致多个系统对于数据库的访问不可达到，这时候可以聚合这些服务器的告警到一条。

  2. **禁止处理**，比如告警提示一个远程数据中心不可达到的情况下，对于数据中心的某一个特殊的机器的访问可能不需要单独的显示出来，因此这种配置有时候可以抑制大量告警充斥的情况发生。

  3. **告警忽略**，简单的阻止任何告警的发生，可以根据配置来匹配不去触发告警的出现。

notifier组建主要用于提供接收规则管理器发送的告警信息，并将告警数据分发到所有配置的告警管理器中进行下一步操作。

#### 1.Alert告警定义

每个告警的数据结构都如下面代码定义，其中Labels包含了告警的所有标签数据（列表结构），每一个标签又包含一个Name,和对应的Value.

<pre><code class="go">&lt;br />// Alert is a generic representation of an alert in the Prometheus eco-system.
type Alert struct {
    // Label value pairs for purpose of aggregation, matching, and disposition
    // dispatching. This must minimally include an "alertname" label.
    Labels labels.Labels `json:"labels"`

    // Extra key/value information which does not define alert identity.
    Annotations labels.Labels `json:"annotations"`

    // The known time range for this alert. Both ends are optional.
    StartsAt     time.Time `json:"startsAt,omitempty"`
    EndsAt       time.Time `json:"endsAt,omitempty"`
    GeneratorURL string    `json:"generatorURL,omitempty"`
}

</code></pre>

#### 2&#46;Notifier定义

Notifier主要负责将告警信息发送到告警管理器中，他的数据结构定义如下，其中Options保存了基本的告警处理的参数信息，比如发送告警到AlertManager的方法和prometheus的注册器

    <br />type Notifier struct {
        queue []*Alert
        opts  *Options
    
        metrics *alertMetrics
    
        more   chan struct{}
        mtx    sync.RWMutex
        ctx    context.Context
        cancel func()
    
        alertmanagers   []*alertmanagerSet
        cancelDiscovery func()
        logger          log.Logger
    }
    

我们在第一部分cmd模块中初始化Notifier对象的时候调用了ApplyConfig方法来加载配置文件，加载的方法在此处定义：

    // ApplyConfig updates the status state as the new config requires.
    func (n *Notifier) ApplyConfig(conf *config.Config) error {
        n.mtx.Lock()
        defer n.mtx.Unlock()
    
        n.opts.ExternalLabels = conf.GlobalConfig.ExternalLabels
        n.opts.RelabelConfigs = conf.AlertingConfig.AlertRelabelConfigs
    
        amSets := []*alertmanagerSet{}
        ctx, cancel := context.WithCancel(n.ctx)
    
        for _, cfg := range conf.AlertingConfig.AlertmanagerConfigs {
            ams, err := newAlertmanagerSet(cfg, n.logger)
            if err != nil {
                return err
            }
    
            ams.metrics = n.metrics
    
            amSets = append(amSets, ams)
        }
    
        // After all sets were created successfully, start them and cancel the
        // old ones.
        for _, ams := range amSets {
            go ams.ts.Run(ctx)
            ams.ts.UpdateProviders(discovery.ProvidersFromConfig(ams.cfg.ServiceDiscoveryConfig, n.logger))
        }
        if n.cancelDiscovery != nil {
            n.cancelDiscovery()
        }
    
        n.cancelDiscovery = cancel
        n.alertmanagers = amSets
    
        return nil
    }
    

#### 3&#46;Notifier处理流程

执行Notifier的Run方法后将启动一个for循环，并等待特殊的执行信号的到达，代码如下,我们在Notifier.Send()方法中可以看到如果有告警产生的时候并非直接发送而是向Notifier.more信道传递信号，使得for循环得以执行调用sendAll方法，

    <br />// Run dispatches notifications continuously.
    func (n *Notifier) Run() {
        for {
            select {
            case <-n.ctx.Done():
                return
            case <-n.more:
            }
            alerts := n.nextBatch()
    
            if !n.sendAll(alerts...) {
                n.metrics.dropped.Add(float64(len(alerts)))
            }
            // If the queue still has items left, kick off the next iteration.
            if n.queueLen() > 0 {
                n.setMore()
            }
        }
    }
    ...
    
    // setMore signals that the alert queue has items.
    func (n *Notifier) setMore() {
        // If we cannot send on the channel, it means the signal already exists
        // and has not been consumed yet.
        select {
        case n.more <- struct{}{}:
        default:
        }
    }
    

sendAll方法中将告警信息进行序列化处理后，获得所有的AlertManager集合，并向每一个集合中定义的url发送消息，这段代码中使用了waitGroup来等待所有的执行的完成。使用统一的context来管理超时的发生

    <br /><br />func (n *Notifier) sendAll(alerts ...*Alert) bool {
        begin := time.Now()
    
        b, err := json.Marshal(alerts)
        if err != nil {
            level.Error(n.logger).Log("msg", "Encoding alerts failed", "err", err)
            return false
        }
    
        n.mtx.RLock()
        amSets := n.alertmanagers
        n.mtx.RUnlock()
    
        var (
            wg         sync.WaitGroup
            numSuccess uint64
        )
        for _, ams := range amSets {
            ams.mtx.RLock()
    
            for _, am := range ams.ams {
                wg.Add(1)
    
                ctx, cancel := context.WithTimeout(n.ctx, ams.cfg.Timeout)
                defer cancel()
    
                go func(ams *alertmanagerSet, am alertmanager) {
                    u := am.url().String()
    
                    if err := n.sendOne(ctx, ams.client, u, b); err != nil {
                        level.Error(n.logger).Log("alertmanager", u, "count", len(alerts), "msg", "Error sending alert", "err", err)
                        n.metrics.errors.WithLabelValues(u).Inc()
                    } else {
                        atomic.AddUint64(&numSuccess, 1)
                    }
                    n.metrics.latency.WithLabelValues(u).Observe(time.Since(begin).Seconds())
                    n.metrics.sent.WithLabelValues(u).Add(float64(len(alerts)))
    
                    wg.Done()
                }(ams, am)
            }
            ams.mtx.RUnlock()
        }
        wg.Wait()
    
        return numSuccess > 0
    }
    

#### 4&#46;告警管理器配置

通过配置可以实现AlertManager的高可用，Prometheus支持同时配置多个AlertManager来管理告警，比如配置文件中我们可以定义：

    alerting:
      alertmanagers:
      - scheme: https
        static_configs:
        - targets:
          - "1.2.3.4:9093"
          - "1.2.3.5:9093"
          - "1.2.3.6:9093"