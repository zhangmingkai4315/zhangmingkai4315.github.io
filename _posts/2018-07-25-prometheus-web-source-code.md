---
id: 86
title: Prometheus源码笔记-web模块
date: 2018-07-25T17:24:42+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=86
permalink: /2018/07/prometheus-web-source-code/
categories:
  - devops
tags:
  - golang
  - prometheus
  - readcode
format: aside
---
## Prometheus源码笔记-web模块

Prometheus的web模块主要提供web相关的操作和federate功能（层次化部署接口），并提供Prometheus的http api接口对外提供服务。

### 1&#46; Handler结构体定义

结构体定义了httpserver启动需要的处理函数，router调用了`github.com/julienschmidt/httprouter`来产生一个router对象，接收所有的http请求并分发路由。其他的组件对象也将传递到handle中提供调用。

    type Handler struct {
        logger log.Logger
    
        targetManager *retrieval.TargetManager
        ruleManager   *rules.Manager
        queryEngine   *promql.Engine
        context       context.Context
        tsdb          func() *tsdb.DB
        storage       storage.Storage
        notifier      *notifier.Notifier
    
        apiV1 *api_v1.API
    
        router       *route.Router
        quitCh       chan struct{}
        reloadCh     chan chan error
        options      *Options
        config       *config.Config
        configString string
        versionInfo  *PrometheusVersion
        birth        time.Time
        cwd          string
        flagsMap     map[string]string
    
        externalLabels model.LabelSet
        mtx            sync.RWMutex
        now            func() model.Time
    
        ready uint32 // ready is uint32 rather than boolean to be able to use atomic functions.
    }
    

#### 2&#46; 初始化router

web处理网络请求开始时候，需要生成一个router用来分发http请求，定义每一个请求的http执行函数

    <br />func New(logger log.Logger, o *Options) *Handler {
        router := route.New()
        cwd, err := os.Getwd()
    
        if err != nil {
            cwd = "<error retrieving current working directory>"
        }
        if logger == nil {
            logger = log.NewNopLogger()
        }
    
        h := &Handler{
            logger:      logger,
            router:      router,
            quitCh:      make(chan struct{}),
            reloadCh:    make(chan chan error),
            options:     o,
            versionInfo: o.Version,
            birth:       time.Now(),
            cwd:         cwd,
            flagsMap:    o.Flags,
    
            context:       o.Context,
            targetManager: o.TargetManager,
            ruleManager:   o.RuleManager,
            queryEngine:   o.QueryEngine,
            tsdb:          o.TSDB,
            storage:       o.Storage,
            notifier:      o.Notifier,
    
            now: model.Now,
    
            ready: 0,
        }
    
        h.apiV1 = api_v1.NewAPI(h.queryEngine, h.storage, h.targetManager, h.notifier,
            func() config.Config {
                h.mtx.RLock()
                defer h.mtx.RUnlock()
                return *h.config
            },
            h.testReady,
            h.options.TSDB,
            h.options.EnableAdminAPI,
        )
    
        ...
    
        router.Get("/alerts", readyf(instrf("alerts", h.alerts)))
        router.Get("/targets", readyf(instrf("targets", h.targets)))
    }
    

下面的代码提供简单的装饰操作，当检测到服务器暂时未就绪的时候，返回503否则进入下一步，该装饰器用于所有路由的处理。

    // Checks if server is ready, calls f if it is, returns 503 if it is not.
    func (h *Handler) testReady(f http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            if h.isReady() {
                f(w, r)
            } else {
                w.WriteHeader(http.StatusServiceUnavailable)
                fmt.Fprintf(w, "Service Unavailable")
            }
        }
    }
    

#### 3&#46;cors支持

通过定义特殊的htto header来允许cors跨域的请求支持，该操作仅仅设置在api路由处理上使用

    var corsHeaders = map[string]string{
        "Access-Control-Allow-Headers":  "Accept, Authorization, Content-Type, Origin",
        "Access-Control-Allow-Methods":  "GET, OPTIONS",
        "Access-Control-Allow-Origin":   "*",
        "Access-Control-Expose-Headers": "Date",
    }
    
    // Enables cross-site script calls.
    func setCORS(w http.ResponseWriter) {
        for h, v := range corsHeaders {
            w.Header().Set(h, v)
        }
    }
    
    ...
        mux.Handle(apiPath+"/", http.StripPrefix(apiPath,
            http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                setCORS(w)
                hhFunc(w, r)
            }),
        ))
    ...
    

#### 4&#46; 静态文件处理

    <br />...
        router.Get("/static/*filepath", instrf("static", h.serveStaticAsset))
    ...
    
    
    func (h *Handler) serveStaticAsset(w http.ResponseWriter, req *http.Request) {
        fp := route.Param(req.Context(), "filepath")
        fp = filepath.Join("web/ui/static", fp)
    
        info, err := ui.AssetInfo(fp)
        if err != nil {
            level.Warn(h.logger).Log("msg", "Could not get file info", "err", err, "file", fp)
            w.WriteHeader(http.StatusNotFound)
            return
        }
        file, err := ui.Asset(fp)
        if err != nil {
            if err != io.EOF {
                level.Warn(h.logger).Log("msg", "Could not get file", "err", err, "file", fp)
            }
            w.WriteHeader(http.StatusNotFound)
            return
        }
    
        http.ServeContent(w, req, info.Name(), info.ModTime(), bytes.NewReader(file))
    }
    

#### 5&#46; Debug处理

当配置了debug选项的时候会启动一些关于debug信息的接口，定义如下：

    <br />...
        router.Get("/debug/*subpath", serveDebug)
        router.Post("/debug/*subpath", serveDebug)
    ...
    
    func serveDebug(w http.ResponseWriter, req *http.Request) {
        ctx := req.Context()
        subpath := route.Param(ctx, "subpath")
    
        if subpath == "/pprof" {
            http.Redirect(w, req, req.URL.Path+"/", http.StatusMovedPermanently)
            return
        }
    
        if !strings.HasPrefix(subpath, "/pprof/") {
            http.NotFound(w, req)
            return
        }
        subpath = strings.TrimPrefix(subpath, "/pprof/")
    
        switch subpath {
        case "cmdline":
            pprof.Cmdline(w, req)
        case "profile":
            pprof.Profile(w, req)
        case "symbol":
            pprof.Symbol(w, req)
        case "trace":
            pprof.Trace(w, req)
        default:
            req.URL.Path = "/debug/pprof/" + subpath
            pprof.Index(w, req)
        }
    }
    

#### 6&#46; HttpServer启动

启动web服务器的代码位于Handler.Run()中，由于函数体内容，此处分别进行讲解，下面代码创建一个监听对象，并设置最大的连接数量，以及通过`github.com/mwitkow/go-conntrack`库来监测当前服务器连接的信息

        listener, err := net.Listen("tcp", h.options.ListenAddress)
        if err != nil {
            return err
        }  
        listener = netutil.LimitListener(listener, h.options.MaxConnections)
    
        // Monitor incoming connections with conntrack.
        listener = conntrack.NewListener(listener,
            conntrack.TrackWithName("http"),
            conntrack.TrackWithTracing())
    

`github.com/cockroachdb/cmux`库用于对于同一个连接上的不同协议的分离，分别创建一个grpcclient的接收处理器，和http请求处理器，以及grpc的服务器。项目代码如下：

        var (
            m       = cmux.New(listener)
            grpcl   = m.Match(cmux.HTTP2HeaderField("content-type", "application/grpc"))
            httpl   = m.Match(cmux.HTTP1Fast())
            grpcSrv = grpc.NewServer()
        )
        av2 := api_v2.New(
            time.Now,
            h.options.TSDB,
            h.options.QueryEngine,
            h.options.Storage.Querier,
            func() []*retrieval.Target {
                return h.options.TargetManager.Targets()
            },
            func() []*url.URL {
                return h.options.Notifier.Alertmanagers()
            },
            h.options.EnableAdminAPI,
        )
        av2.RegisterGRPC(grpcSrv)
       ...
    
        httpSrv := &http.Server{
            Handler:     nethttp.Middleware(opentracing.GlobalTracer(), mux, operationName),
            ErrorLog:    errlog,
            ReadTimeout: h.options.ReadTimeout,
        }
    
        go func() {
            if err := httpSrv.Serve(httpl); err != nil {
                level.Warn(h.logger).Log("msg", "error serving HTTP", "err", err)
            }
        }()
        go func() {
            if err := grpcSrv.Serve(grpcl); err != nil {
                level.Warn(h.logger).Log("msg", "error serving gRPC", "err", err)
            }
        }()
    
        ...
        go func() {
            errCh <- m.Serve()
        }()
    

Run函数当接收到错误或者context对象被终止后均会导致程序的退出

        select {
        case e := <-errCh:
            return e
        case <-ctx.Done():
            httpSrv.Shutdown(ctx)
            grpcSrv.GracefulStop()
            return nil
        }
    

#### 7&#46; AlertStatus及处理路由

定义一个基本的告警状态的结构体以及满足排序接口的结构体对象，并针对/alert路由请求返回包含所有告警信息的页面。

    type AlertStatus struct {
        AlertingRules        []*rules.AlertingRule
        AlertStateToRowClass map[rules.AlertState]string
    }
    
    type byAlertStateAndNameSorter struct {
        alerts []*rules.AlertingRule
    }
    
    func (s byAlertStateAndNameSorter) Len() int {
        return len(s.alerts)
    }
    
    func (s byAlertStateAndNameSorter) Less(i, j int) bool {
        return s.alerts[i].State() > s.alerts[j].State() ||
            (s.alerts[i].State() == s.alerts[j].State() &&
                s.alerts[i].Name() < s.alerts[j].Name())
    }
    
    func (s byAlertStateAndNameSorter) Swap(i, j int) {
        s.alerts[i], s.alerts[j] = s.alerts[j], s.alerts[i]
    }
    
    
    ...
    router.Get("/alerts", readyf(instrf("alerts", h.alerts)))
    ...
    
    func (h *Handler) alerts(w http.ResponseWriter, r *http.Request) {
        alerts := h.ruleManager.AlertingRules()
        alertsSorter := byAlertStateAndNameSorter{alerts: alerts}
        sort.Sort(alertsSorter)
    
        alertStatus := AlertStatus{
            AlertingRules: alertsSorter.alerts,
            AlertStateToRowClass: map[rules.AlertState]string{
                rules.StateInactive: "success",
                rules.StatePending:  "warning",
                rules.StateFiring:   "danger",
            },
        }
        h.executeTemplate(w, "alerts.html", alertStatus)
    }
    
    
    // AlertStatus bundles alerting rules and the mapping of alert states to row classes.
    

#### 8&#46; API接口定义

api目录中包含了v1和v2两个文件夹来包含不同版本的api接口，这些接口通过Register函数注册到http的路由对象中，代码如下：

    // Register the API's endpoints in the given router.
    func (api *API) Register(r *route.Router) {
        instr := func(name string, f apiFunc) http.HandlerFunc {
            hf := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                setCORS(w)
                if data, err := f(r); err != nil {
                    respondError(w, err, data)
                } else if data != nil {
                    respond(w, data)
                } else {
                    w.WriteHeader(http.StatusNoContent)
                }
            })
            return api.ready(prometheus.InstrumentHandler(name, httputil.CompressionHandler{
                Handler: hf,
            }))
        }
    
        r.Options("/*path", instr("options", api.options))
    
        r.Get("/query", instr("query", api.query))
        r.Post("/query", instr("query", api.query))
        r.Get("/query_range", instr("query_range", api.queryRange))
        r.Post("/query_range", instr("query_range", api.queryRange))
        ...
    }
    

对于其中的query的处理，则通过queryEngine来获得查询的结果返回给，首先定义一个上下文，并设置超时的时间，当查询的对象其中定义了几个错误状态来分别处理不同情况下的解析错误，比如超时，或者存储错误等如果没有错误则将查询的数据返回给

        ctx := r.Context()
        if to := r.FormValue("timeout"); to != "" {
            var cancel context.CancelFunc
            timeout, err := parseDuration(to)
            if err != nil {
                return nil, &apiError{errorBadData, err}
            }
    
            ctx, cancel = context.WithTimeout(ctx, timeout)
            defer cancel()
        }
    
        qry, err := api.QueryEngine.NewInstantQuery(r.FormValue("query"), ts)
        if err != nil {
            return nil, &apiError{errorBadData, err}
        }
    
        res := qry.Exec(ctx)
        if res.Err != nil {
            switch res.Err.(type) {
            case promql.ErrQueryCanceled:
                return nil, &apiError{errorCanceled, res.Err}
            case promql.ErrQueryTimeout:
                return nil, &apiError{errorTimeout, res.Err}
            case promql.ErrStorage:
                return nil, &apiError{errorInternal, res.Err}
            }
            return nil, &apiError{errorExec, res.Err}
        }
    
        // Optional stats field in response if parameter "stats" is not empty.
        var qs *stats.QueryStats
        if r.FormValue("stats") != "" {
            qs = stats.NewQueryStats(qry.Stats())
        }
    
        return &queryData{
            ResultType: res.Value.Type(),
            Result:     res.Value,
            Stats:      qs,
        }, nil
    

标准的api响应信息通过respondError函数来封装，只需要传递一个apiErr对象以及需要传递的数据即可,通过类型的type来设定不同的状态code

    func respondError(w http.ResponseWriter, apiErr *apiError, data interface{}) {
        w.Header().Set("Content-Type", "application/json")
    
        var code int
        switch apiErr.typ {
        case errorBadData:
            code = http.StatusBadRequest
        case errorExec:
            code = 422
        case errorCanceled, errorTimeout:
            code = http.StatusServiceUnavailable
        case errorInternal:
            code = http.StatusInternalServerError
        default:
            code = http.StatusInternalServerError
        }
        w.WriteHeader(code)
    
        b, err := json.Marshal(&response{
            Status:    statusError,
            ErrorType: apiErr.typ,
            Error:     apiErr.err.Error(),
            Data:      data,
        })
        if err != nil {
            return
        }
        w.Write(b)
    }