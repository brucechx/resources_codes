[TOC]

### Distributor

distributor` 直接接收来自 `promtail 的日志写入请求, 请求体由 protobuf 编码, 格式如下:

```go
// 一次写入请求, 包含多段日志流
type PushRequest struct {
    Streams []*Stream `protobuf:"bytes,1,rep,name=streams" json:"streams,omitempty"`
}
// 一段日志流, 包含它的 label, 以及这段日志流当中的每个日志事件: Entry
type Stream struct {
    Labels  string  `protobuf:"bytes,1,opt,name=labels,proto3" json:"labels,omitempty"`
    Entries []Entry `protobuf:"bytes,2,rep,name=entries" json:"entries"`
}
// 一个日志事件, 包含时间戳与内容
type Entry struct {
    Timestamp time.Time `protobuf:"bytes,1,opt,name=timestamp,stdtime" json:"timestamp"`
    Line      string    `protobuf:"bytes,2,opt,name=line,proto3" json:"line,omitempty"`
}
```

#### Distributor 上报的日志流样式

```json
{
  "streams": [
    {
      "labels": "logs{__filename__=\"/data/log/umonitor3/alarmp/alarmp201905130.log\", job=\"alarmp\", target=\"172.18.178.244\"}",
      "entries": [
        {
          "ts": "2019-05-13T12:57:19.813801543+08:00",
          "line": "[2019-05-13 10:45:30.693572] [INFO][model][update] UsrGroupMap the 2019-05-13 10:44:57 time, UsrGroupMap update count=1, total count: 15"
        }
        ]
    }
  ]
}
```

#### 

`distributor` 收到请求后, 会将一个 `PushRequest` 中的 `Stream` 根据 labels 拆分成多个 `PushRequest`, 这个过程使用一致性哈希:

```go
// First we flatten out the request into a list of samples.
	// We use the heuristic of 1 sample per TS to size the array.
	// We also work out the hash value at the same time.
	streams := make([]streamTracker, 0, len(req.Streams))
	keys := make([]uint32, 0, len(req.Streams))
	var validationErr error
	for _, stream := range req.Streams {
		if err := d.validateLabels(userID, stream.Labels); err != nil {
			validationErr = err
			continue
		}
		
        // 获取每个 stream 的 label hash
		keys = append(keys, tokenFor(userID, stream.Labels))
		streams = append(streams, streamTracker{
			stream: stream,
		})
	}

	if len(streams) == 0 {
		return &logproto.PushResponse{}, validationErr
	}

	// 根据 label hash 到 hash ring 上获取对应的 ingester 节点
	// 这里的节点指 hash ring 上的节点, 一个节点可能有多个对等的 ingester 副本来做 HA
	replicationSets, err := d.ring.BatchGet(keys, ring.Write)
	if err != nil {
		return nil, err
	}
	// 将 Stream 按对应的 ingester 节点进行分组
	samplesByIngester := map[string][]*streamTracker{}
	ingesterDescs := map[string]ring.IngesterDesc{}
	for i, replicationSet := range replicationSets {
		streams[i].minSuccess = len(replicationSet.Ingesters) - replicationSet.MaxErrors
		streams[i].maxFailures = replicationSet.MaxErrors
		for _, ingester := range replicationSet.Ingesters {
			samplesByIngester[ingester.Addr] = append(samplesByIngester[ingester.Addr], &streams[i])
			ingesterDescs[ingester.Addr] = ingester
		}
	}

	tracker := pushTracker{
		samplesPending: int32(len(streams)),
		done:           make(chan struct{}),
		err:            make(chan error),
	}
	for ingester, samples := range samplesByIngester {
        // 每组 Stream[] 又作为一个 PushRequest, 下发给对应的 ingester 节点
		go func(ingester ring.IngesterDesc, samples []*streamTracker) {
			// Use a background context to make sure all ingesters get samples even if we return early
			localCtx, cancel := context.WithTimeout(context.Background(), d.clientCfg.RemoteTimeout)
			defer cancel()
			localCtx = user.InjectOrgID(localCtx, userID)
			if sp := opentracing.SpanFromContext(ctx); sp != nil {
				localCtx = opentracing.ContextWithSpan(localCtx, sp)
			}
			d.sendSamples(localCtx, ingester, samples, &tracker)
		}(ingesterDescs[ingester], samples)
	}
```

### 