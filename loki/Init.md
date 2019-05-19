[TOC]

**概述**：Loki 整体分为七个模块 Ring Overrides Server Distributor Ingester  Querier Store；默认配置是All，即七个模块全部加载；

**整体流程**： 主入口是cmd pkg 下的main.go 文件；

1. 加载config，配置log, trace 等

2. loki 初始化

   ```go
   t, err := loki.New(cfg)
   ```

   - 依赖模块初始化

     ```go
     if err := loki.init(cfg.Target)  // cfg.Target 默认为All 
     ```

3. loki 运行

   ```go
   if err := t.Run();
   ```

   - 启动server 进程

     ```go
     // Run starts Loki running, and blocks until a signal is received.
     func (t *Loki) Run() error {
     	return t.server.Run()
     }
     ```

## 依赖模块初始化

### Server

使用weave works 依赖包中的server 初始化，启动tcp 和 grpc 服务

### Distributor

```go
// 负责对接promtail 日志采集
func (t *Loki) initDistributor() (err error) {
	t.distributor, err = distributor.New(t.cfg.Distributor, t.cfg.IngesterClient, t.ring, t.overrides)
	if err != nil {
		return
	}
	// 处理promtail 上传而来的日志流
	t.server.HTTP.Handle("/api/prom/push", middleware.Merge(
		t.httpAuthMiddleware,
	).Wrap(http.HandlerFunc(t.distributor.PushHandler)))

	return
}
```



### Ingester

```go
// 负责对接存储
func (t *Loki) initIngester() (err error) {
	t.cfg.Ingester.LifecyclerConfig.ListenPort = &t.cfg.Server.GRPCListenPort
	t.ingester, err = ingester.New(t.cfg.Ingester, t.store)
	if err != nil {
		return
	}
 	// 注册grpc 服务
	logproto.RegisterPusherServer(t.server.GRPC, t.ingester)
	logproto.RegisterQuerierServer(t.server.GRPC, t.ingester)
	grpc_health_v1.RegisterHealthServer(t.server.GRPC, t.ingester)
	// 注册http 服务 	  		  t.server.HTTP.Path("/ready").Handler(http.HandlerFunc(t.ingester.ReadinessHandler))
	t.server.HTTP.Path("/flush").Handler(http.HandlerFunc(t.ingester.FlushHandler))
	return
}
```

```go
// New makes a new Ingester.
func New(cfg Config, store ChunkStore) (*Ingester, error) {
	i := &Ingester{
		cfg:         cfg,
		instances:   map[string]*instance{},
		store:       store,
		quit:        make(chan struct{}),
		flushQueues: make([]*util.PriorityQueue, cfg.ConcurrentFlushes),
	}

	i.flushQueuesDone.Add(cfg.ConcurrentFlushes)
  // 默认值是16个，实时消费日志流
	for j := 0; j < cfg.ConcurrentFlushes; j++ {
		i.flushQueues[j] = util.NewPriorityQueue(flushQueueLength)
		go i.flushLoop(j)  // 消费队列
	}

	var err error
    // 注册到一致性hash环
	i.lifecycler, err = ring.NewLifecycler(cfg.LifecyclerConfig, i)
	if err != nil {
		return nil, err
	}

	i.done.Add(1)
	go i.loop() // 默认30s定时消费和检查，清空无日志流的索引

	return i, nil
}
```



### Querier

```go
// 负责对接查询
func (t *Loki) initQuerier() (err error) {
	t.querier, err = querier.New(t.cfg.Querier, t.cfg.IngesterClient, t.ring, t.store)
	if err != nil {
		return
	}

	httpMiddleware := middleware.Merge(
		t.httpAuthMiddleware,
	)
	// 支持的http接口
	t.server.HTTP.Handle("/api/prom/query", httpMiddleware.Wrap(http.HandlerFunc(t.querier.QueryHandler)))
	t.server.HTTP.Handle("/api/prom/label", httpMiddleware.Wrap(http.HandlerFunc(t.querier.LabelHandler)))
	t.server.HTTP.Handle("/api/prom/label/{name}/values", httpMiddleware.Wrap(http.HandlerFunc(t.querier.LabelHandler)))
	return
}
```



### Store

```go
// 使用 cortex 框架的storage模块，初始化chunk存储
func (t *Loki) initStore() (err error) {
	t.store, err = storage.NewStore(t.cfg.StorageConfig, t.cfg.ChunkStoreConfig, t.cfg.SchemaConfig, t.overrides)
	return
}
```



### Ring

```go
// 使用 cortex 框架的ring模块，一致性hash环
func (t *Loki) initRing() (err error) {
	t.ring, err = ring.New(t.cfg.Ingester.LifecyclerConfig.RingConfig)
	if err != nil {
		return
	}
	t.server.HTTP.Handle("/ring", t.ring)
	return
}
```



### Overrides

```go
// 使用 cortex 框架的overrides模块, 用作校验
func (t *Loki) initOverrides() (err error) {
	t.overrides, err = validation.NewOverrides(t.cfg.LimitsConfig)
	return err
}
```

