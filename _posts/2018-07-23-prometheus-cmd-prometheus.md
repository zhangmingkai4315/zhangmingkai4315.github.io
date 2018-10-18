---
id: 41
title: Prometheus源码笔记-cmd/prometheus模块
date: 2018-07-23T11:32:41+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=41
permalink: /2018/07/prometheus-cmd-prometheus/
categories:
  - devops
tags:
  - golang
  - prometheus
  - readcode
  - 监控系统
---
`cmd/prometheus`模块作为程序的入口主要执行程序的初始化和组件的启动工作，核心代码位于`cmd/prometheus/main.go`中。本节主要是对于该模块的阅读记录，并对于核心处理逻辑进行解释。另外Prometheus的源码遵从Apache2.0许可协议，也就是如果用于商业目的的话也不需要像GNU许可协议软件那样遵从强制开源要求。

### cmd/prometheus模块

#### 1&#46; 初始化操作

除了执行import引用其他模块中的初始化init()函数外，程序本身的初始化代码如下分别创建了两个Gauge对象（接口类型），这两个对象主要用于程序配置文件状态监控，保存上一次载入配置文件成功的状态数据,这些数据也会出现在监控中。 执行version信息的注册（提供当前程序的版本信息等）。

<pre><code class="go">var (
    configSuccess = prometheus.NewGauge(prometheus.GaugeOpts{
        Namespace: "prometheus",
        Name:      "config_last_reload_successful",
        Help:      "Whether the last configuration reload attempt was successful.",
    })
    configSuccessTime = prometheus.NewGauge(prometheus.GaugeOpts{
        Namespace: "prometheus",
        Name:      "config_last_reload_success_timestamp_seconds",
        Help:      "Timestamp of the last successful configuration reload.",
    })
)

func init() {
    prometheus.MustRegister(version.NewCollector("prometheus"))
}

...

    prometheus.MustRegister(configSuccess)
    prometheus.MustRegister(configSuccessTime)

</code></pre>

#### 2&#46; 主程序main()

##### 2&#46;1 参数及配置管理

主程序中有接近一半代码用于程序的参数处理，这里使用了`"gopkg.in/alecthomas/kingpin.v2"`模块来处理程序的传递参数，该模块相对于原生flag提供更加灵活的参数处理，命令及所属组管理。初始化的配置对象结构体如下：

<pre><code class="go">    cfg := struct {
        configFile string

        localStoragePath string
        notifier         notifier.Options
        notifierTimeout  model.Duration
        queryEngine      promql.EngineOptions
        web              web.Options
        tsdb             tsdb.Options
        lookbackDelta    model.Duration
        webTimeout       model.Duration
        queryTimeout     model.Duration

        prometheusURL string

        logLevel promlog.AllowedLevel
    }{
        notifier: notifier.Options{
            Registerer: prometheus.DefaultRegisterer,
        },
        queryEngine: promql.EngineOptions{
            Metrics: prometheus.DefaultRegisterer,
        },
    }
</code></pre>

初始化过程中传递了缺省的notifier和queryEngine分别用于告警模块和prom ql语句处理引擎的加载, notifer中的Registerer加载缺省的Registerer,包含一个go collector和一个process collector。

##### 2&#46;2 初始化存储对象

结构体初始化后，完成存储对象的初始化，这些对象用于管理存储时间序列的数据，首先初始化本地存储管理组件localStorage，远程存储remoteStorage用于比如Graphite, OpenTSDB或者 InfluxDB等与Prometheus的结合使用,fanoutStorage主要是定义一个读写代理器，来代理本地存储和远程存储之间的数据的读写操作。

<pre><code class="go">    var (
        localStorage  = &tsdb.ReadyStorage{}
        remoteStorage = remote.NewStorage(log.With(logger, "component", "remote"), localStorage.StartTime)
        fanoutStorage = storage.NewFanout(logger, localStorage, remoteStorage)
    )
</code></pre>

本地存储的数据缺省每隔2小时形成一个block，每一个block包含一个目录里面存储了所有这段时间内的时序数据，这样数据如果出现问题，只需要移除该2小时的block即可，不需要删除整个存储数据。缺省删除存储旧数据的时间是15天，缺省存储位置位于data/下面，对于存储数据空间需求一般可以按照如下计算：

    needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample(一般1-2字节)
    

##### 2&#46;3 初始化核心组件

利用其他模块来产生一些组建对象，比如notifer组建，解释引擎组建，及规则管理器。

        var (
            notifier       = notifier.New(&cfg.notifier, log.With(logger, "component", "notifier"))
            targetManager  = retrieval.NewTargetManager(fanoutStorage, log.With(logger, "component", "target manager"))
            queryEngine    = promql.NewEngine(fanoutStorage, &cfg.queryEngine)
            ctx, cancelCtx = context.WithCancel(context.Background())
        )
    
        ruleManager := rules.NewManager(&rules.ManagerOptions{
            Appendable:  fanoutStorage,
            QueryFunc:   rules.EngineQueryFunc(queryEngine),
            NotifyFunc:  sendAlerts(notifier, cfg.web.ExternalURL.String()),
            Context:     ctx,
            ExternalURL: cfg.web.ExternalURL,
            Registerer:  prometheus.DefaultRegisterer,
            Logger:      log.With(logger, "component", "rule manager"),
        })
    

##### 2&#46;4 任务执行组管理

借助于`"github.com/oklog/oklog/pkg/group"`模块中的goroutine组管理，可以提供对于加入到执行队列中的goroutine执行流程的管理，保证goroutine的正确执行和退出。分别完成信号绑定处理函数，启动tsdb,启动webserver,启动规则处理，启动告警处理，启动目标对象管理，一旦任何程序出现退出，则发送信号告知退出主程序。

        var g group.Group
        {
            term := make(chan os.Signal)
            signal.Notify(term, os.Interrupt, syscall.SIGTERM)
            cancel := make(chan struct{})
            g.Add(
                func() error {
                    select {
                    case <-term:
                        level.Warn(logger).Log("msg", "Received SIGTERM, exiting gracefully...")
                    case <-webHandler.Quit():
                        level.Warn(logger).Log("msg", "Received termination request via web service, exiting gracefully...")
                    case <-cancel:
                        break
                    }
                    return nil
                },
                func(err error) {
                    close(cancel)
                },
            )
        }
        ...
    

##### 2&#46;5 辅助函数

载入配置文件，注意defer中传递新的配置是否成功的数据及时间到监测数据集合中。其中参数rls（reloaders）保存了config对象的配置函数列表，将config对象传递到相关组件中，函数及reloaders如下：

    func reloadConfig(filename string, logger log.Logger, rls ...func(*config.Config) error) (err error) {
        level.Info(logger).Log("msg", "Loading configuration file", "filename", filename)
    
        defer func() {
            if err == nil {
                configSuccess.Set(1)
                configSuccessTime.Set(float64(time.Now().Unix()))
            } else {
                configSuccess.Set(0)
            }
        }()
    
        conf, err := config.LoadFile(filename)
        if err != nil {
            return fmt.Errorf("couldn't load configuration (--config.file=%s): %v", filename, err)
        }
    
        failed := false
        for _, rl := range rls {
            if err := rl(conf); err != nil {
                level.Error(logger).Log("msg", "Failed to apply configuration", "err", err)
                failed = true
            }
        }
        if failed {
            return fmt.Errorf("one or more errors occurred while applying the new configuration (--config.file=%s)", filename)
        }
        return nil
    }
    
    
    reloaders := []func(cfg *config.Config) error{
            remoteStorage.ApplyConfig,
            targetManager.ApplyConfig,
            webHandler.ApplyConfig,
            notifier.ApplyConfig,
            func(cfg *config.Config) error {
                // Get all rule files matching the configuration oaths.
                var files []string
                for _, pat := range cfg.RuleFiles {
                    fs, err := filepath.Glob(pat)
                    if err != nil {
                        // The only error can be a bad pattern.
                        return fmt.Errorf("error retrieving rule files for %s: %s", pat, err)
                    }
                    files = append(files, fs...)
                }
                return ruleManager.Update(time.Duration(cfg.GlobalConfig.EvaluationInterval), files)
            },
        }
    

根据实际配置中url返回告警处理函数，并对于规则进行过滤，找到需要发送alert的数据并调用Notifier对象的send函数发送出去。

    // sendAlerts implements a the rules.NotifyFunc for a Notifier.
    // It filters any non-firing alerts from the input.
    func sendAlerts(n *notifier.Notifier, externalURL string) rules.NotifyFunc {
        return func(ctx context.Context, expr string, alerts ...*rules.Alert) error {
            var res []*notifier.Alert
    
            for _, alert := range alerts {
                // Only send actually firing alerts.
                if alert.State == rules.StatePending {
                    continue
                }
                a := &notifier.Alert{
                    StartsAt:     alert.FiredAt,
                    Labels:       alert.Labels,
                    Annotations:  alert.Annotations,
                    GeneratorURL: externalURL + strutil.TableLinkForExpression(expr),
                }
                if !alert.ResolvedAt.IsZero() {
                    a.EndsAt = alert.ResolvedAt
                }
                res = append(res, a)
            }
    
            if len(alerts) > 0 {
                n.Send(res...)
            }
            return nil
        }
    }
    

FdLimits返回当前系统的文件描述符的限制（仅仅适用于非windows系统）

    // FdLimits returns the soft and hard limits for file descriptors
    func FdLimits() string {
        flimit := syscall.Rlimit{}
        err := syscall.Getrlimit(syscall.RLIMIT_NOFILE, &flimit)
        if err != nil {
            log.Fatal("Error!")
        }
        return fmt.Sprintf("(soft=%d, hard=%d)", flimit.Cur, flimit.Max)
    }
    

### cmd/promtool

程序主要用于提供系统运行的工具，比如检查配置文件，规则等的有效性及提供一些规则和数据的检测，比如：

    examples:
    
    $ cat metrics.prom | promtool check metrics
    
    $ curl -s http://localhost:9090/metrics | promtool check metrics
    

用于检测的代码如下：

    // CheckMetrics performs a linting pass on input metrics.
    func CheckMetrics() int {
        l := promlint.New(os.Stdin)
        problems, err := l.Lint()
        if err != nil {
            fmt.Fprintln(os.Stderr, "error while linting:", err)
            return 1
        }
    
        for _, p := range problems {
            fmt.Fprintln(os.Stderr, p.Metric, p.Text)
        }
    
        if len(problems) > 0 {
            return 3
        }
    
        return 0
    }
    

其他检测函数此处就不一一解释了，cmd模块的工作及流程就包含这些。