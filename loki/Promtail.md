[TOC]

# Promtail

通过服务发现确定要收集的应用以及应用的日志路径后, `promtail` 就开始了真正的日志收集过程. 这里分三步:

1. 用 `fsnotify` 监听对应目录下的文件创建与删除(处理 log rolling)
2. 对每个活跃的日志文件起一个 goroutine 进行类似 `tail -f` 的读取, 读取到的内容发送给 `channel`
3. 一个单独的 goroutine 会解析 `channel` 中的日志行, 分批发送给 loki 的 backend

```go
// New makes a new Promtail.
func New(cfg config.Config) (*Promtail, error) {
	// 写入读取文件的偏移量，后台启动
	positions, err := positions.New(util.Logger, cfg.PositionsConfig)
	if err != nil {
		return nil, err
	}
	// 收集日志流，达到一定大小batch 后，上传给distributor， 后台启动
	client, err := client.New(cfg.ClientConfig, util.Logger)
	if err != nil {
		return nil, err
	}
	// 日志采集主模块， 后台启动
	tms, err := targets.NewTargetManagers(util.Logger, positions, client, cfg.ScrapeConfig, &cfg.TargetConfig)
	if err != nil {
		return nil, err
	}
	// grpc 和 http 服务，阻塞主进程，单独启动
	server, err := server.New(cfg.ServerConfig)
	if err != nil {
		return nil, err
	}

	return &Promtail{
		client:         client,
		positions:      positions,
		targetManagers: tms,
		server:         server,
	}, nil
}
```

