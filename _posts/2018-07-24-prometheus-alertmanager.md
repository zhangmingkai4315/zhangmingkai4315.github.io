---
layout: post
title: Prometheus源码笔记-AlertManager
date: 2018-07-24T11:32:41+00:00
author: mingkai
permalink: /2018/07/prometheus-with-alertmanager/
categories:
  - devops
tags:
  - devops
---

AlertManager 用于接收 Prometheus 发送的告警并对于告警进行一系列的处理后发送给指定的用户。系统的整体设计图如下面所示，并且支持 HA 高可用部署。

![alertmanager](https://github.com/prometheus/alertmanager/raw/master/doc/arch.svg?sanitize=true)

Prometheus 或者告警发送系统可以通过 API 的方式发送给 Alertmanager，收到告警后将告警分别存储在 AlertProvider 中（当前实现是存储在内存中，可以通过接口的方式自行实现其他存储方式比如 MySQL 或者 ES）。

```golang
# api/api.go

	r.Get("/alerts", wrap(api.listAlerts))
	r.Post("/alerts", wrap(api.addAlerts))

func (api *API) insertAlerts(w http.ResponseWriter, r *http.Request, alerts ...*types.Alert) {
     ...

	for _, a := range alerts {
		removeEmptyLabels(a.Labels)
		if err := a.Validate(); err != nil {
			validationErrs.Add(err)
			numInvalidAlerts.Inc()
			continue
		}
		validAlerts = append(validAlerts, a)
	}
	if err := api.alerts.Put(validAlerts...); err != nil {
		api.respondError(w, apiError{
			typ: errorInternal,
			err: err,
		}, nil)
		return
	}
    ...
}
```

AlertManager 内部的 Dispatcher 通过订阅的方式获得告警信息更新（获得 Alerts 的迭代器，通过 for 循环不断的获得发送到信道中的 Alerts,通过 route 的 match 函数获得匹配的 route 对象（比如基于标签的正则表达，传递到不同的邮件或者 slack 信道路由）,并且每隔一段时间将执行一次清理操作（当 ag 中的告警数量为空的时候），删除之前的记录。收到的 Alert 通过标签匹配的方式被送到不同的聚合组中等待 Pipeline 流程进行处理。

```golang
func (d *Dispatcher) run(it provider.AlertIterator) {
    ...
	for {
		select {
		case alert, ok := <-it.Next():
			if !ok {
				// Iterator exhausted for some reason.
			...
			for _, r := range d.route.Match(alert.Labels) {
				d.processAlert(alert, r)
			}

		case <-cleanup.C:
			d.mtx.Lock()

			for _, groups := range d.aggrGroups {
				for _, ag := range groups {
					if ag.empty() {
						ag.stop()
						delete(groups, ag.fingerprint())
					}
				}
			}

			d.mtx.Unlock()

		case <-d.ctx.Done():
			return
		}
	}
}
```

聚合组用来管理具有相同属性信息的告警，通过将相同类型的告警进行分组可以统一的管理，因为有时候告警处理是大量同时出现的（比如一个数据中心的失效将导致成百上千的告警产生，通过分组可以聚合相同标签到一个邮件或者接收者中）。分组创建将依赖于处理 route 路由和告警的 labels 标签，不同的告警 labels 将产生不同的聚合组，所有接受到的告警将首先计算一个聚合组的 Fingerprint 如果找到则直接插入到该组，否则创建一个新的聚合组，每次新创建的聚合组都会启动一个 goroutine 来执行实际的 pipeline work.

```golang
# dispatch/dispatch.go
# processAlert()

	groupLabels := model.LabelSet{}

	for ln, lv := range alert.Labels {
		if _, ok := route.RouteOpts.GroupBy[ln]; ok {
			groupLabels[ln] = lv
		}
	}

	fp := groupLabels.Fingerprint()

	d.mtx.Lock()
	group, ok := d.aggrGroups[route]
	if !ok {
		group = map[model.Fingerprint]*aggrGroup{}
		d.aggrGroups[route] = group
	}
	d.mtx.Unlock()
```

每一个聚合组在管理告警上都会通过内部的 run 方法来不断的循环获取一段时间内的告警，并对于时间段的告警进行聚合处理，如下面的代码所示，当接收到完成信号时候退出 run 方法结束该组。默认的 GroupInterval 时间为 5 分钟

```golang
func (ag *aggrGroup) run(nf notifyFunc) {
	ag.done = make(chan struct{})
	defer close(ag.done)
	defer ag.next.Stop()

	for {
		select {
		case now := <-ag.next.C:
			// Give the notifications time until the next flush to
			// finish before terminating them.
			ctx, cancel := context.WithTimeout(ag.ctx, ag.timeout(ag.opts.GroupInterval))
            ...

			ag.flush(func(alerts ...*types.Alert) bool {
				return nf(ctx, alerts...)
			})

			cancel()

		case <-ag.ctx.Done():
			return
		}
	}
}
```

执行的 flush 函数将首先对于 alerts 进行排序（依赖于 job 和 instance）,排序后的 alerts 组将被传递给 notify 函数进行处理

## Pipeline

Pipeline 用来定义告警处理流程，Alertmanager 当前对于告警处理支持的流程包括：Inhibitor, Silencer。

### Inhibit 管理

Inhibitor 用于管理相同的告警配置，比如下面的配置定义了当告警名称 alertname 一致的时候，如果严重告警存在的时候，途同级别告警将被过滤掉。

```yaml
inhibit_rules:
  - source_match:
      severity: "critical"
    target_match:
      severity: "warning"
    # Apply inhibition if the alertname is the same.
    equal: ["alertname"]
```

查询流程上将获得的 alert 的 label 进行检查，匹配检查的内容满足 target 匹配但是 source 不匹配的标记为 Inhibited.

```golang
// Mutes returns true iff the given label set is muted.
func (ih *Inhibitor) Mutes(lset model.LabelSet) bool {
	fp := lset.Fingerprint()

	for _, r := range ih.rules {
		// Only inhibit if target matchers match but source matchers don't.
		if inhibitedByFP, eq := r.hasEqual(lset); !r.SourceMatchers.Match(lset) && r.TargetMatchers.Match(lset) && eq {
			ih.marker.SetInhibited(fp, inhibitedByFP.String())
			return true
		}
	}
	ih.marker.SetInhibited(fp)

	return false
}
```

其中的 inhibited.marker 是一个结构体对象实现了 Marker 接口,结构对象定义如下,通过这个接口实现，可以用来管理告警状态比如设置 Inhibited 和 Silieced 状态，获取统计信息和按状态列出指定的类别告警：

```golang
// Marker helps to mark alerts as silenced and/or inhibited.
// All methods are goroutine-safe.
type Marker interface {
	SetActive(alert model.Fingerprint)
	SetInhibited(alert model.Fingerprint, ids ...string)
	SetSilenced(alert model.Fingerprint, ids ...string)

	Count(...AlertState) int

	Status(model.Fingerprint) AlertStatus
	Delete(model.Fingerprint)

	Unprocessed(model.Fingerprint) bool
	Active(model.Fingerprint) bool
	Silenced(model.Fingerprint) ([]string, bool)
	Inhibited(model.Fingerprint) ([]string, bool)
}
```

### Silence 管理

Silencer 用来取消告警，比如直接配置告警在某一段时间内不触发任何消息，可以基于正则表达式的匹配，该配置可以通过 alertmanager 的 WebUI 或者 API 接口配置。

下面的代码是 Pipeline 中执行的过程，当流程传递到 Silence 步骤时候，Silence 模块将循环检查每一个告警是否满足匹配，比如设置某一个告警标签出现后取消告警。当查询结束后返回一个 sils（Silence 的结构体，用来指定某一类告警的 Silence 在一段时间内的处理对象。）一个告警可能会被多个 Silence 同时管理。

```golang
func (n *SilenceStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var filtered []*types.Alert
	for _, a := range alerts {
		// TODO(fabxc): increment total alerts counter.
		// Do not send the alert if the silencer mutes it.
		sils, err := n.silences.Query(
			silence.QState(types.SilenceStateActive),
			silence.QMatches(a.Labels),
		)
		if err != nil {
			level.Error(l).Log("msg", "Querying silences failed", "err", err)
		}

		if len(sils) == 0 {
			// TODO(fabxc): increment muted alerts counter.
			filtered = append(filtered, a)
			n.marker.SetSilenced(a.Labels.Fingerprint())
		} else {
			ids := make([]string, len(sils))
			for i, s := range sils {
				ids[i] = s.Id
			}
			n.marker.SetSilenced(a.Labels.Fingerprint(), ids...)
		}
	}

	return ctx, filtered, nil
}
```

同时要实现集群管理，彼此之间的 Silence 状态也要共享（告警发送给多个 AM），因此系统设计的时候加入了 SilenceProvider 来进行集群之间的 Silence 管理,彼此之间通过 protoBuf 来进行数据状态的同步。同时集群在接收到告警后也要进行通知，告知其他的节点关于告警的处理状态，防止多个通知同时被发送。

##### (未完)
