[TOC]

##Querier

### 概述

最后是 `Querier`, 这个比较简单了, 大致逻辑就是根据`chunk index`中的索引信息, 请求 `ingester` 和对象存储. 合并后返回;

`loki` 里用了堆, 时间正序就用最小堆, 时间逆序就用最大堆:

```go
// 这部分代码实现了一个简单的二叉堆, MinHeap 和 MaxHeap 实现了相反的 `Less()` 方法
// EntryIterator iterates over entries in time-order.
type EntryIterator interface {
	Next() bool
	Entry() logproto.Entry
	Labels() string
	Error() error
	Close() error
}
```

###livetail 分析

```bash
./logcli --addr="http://192.168.150.25:3100" query -t '{job="varlogs"}'
```

#### TailHandler 服务端websocket 接口

```go
// TailHandler is a http.HandlerFunc for handling tail queries.
func (q *Querier) TailHandler(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		CheckOrigin: func(r *http.Request) bool { return true },
	}

	queryRequestPtr, err := httpRequestToQueryRequest(r)
	if err != nil {
		server.WriteError(w, err)
		return
	}

	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		level.Error(util.Logger).Log("Error in upgrading websocket", fmt.Sprintf("%v", err))
		return
	}

	defer func() {
		err := conn.Close()
		level.Error(util.Logger).Log("Error closing websocket", fmt.Sprintf("%v", err))
	}()

	// response from httpRequestToQueryRequest is a ptr, if we keep passing pointer down the call then it would stay on
	// heap until connection to websocket stays open
	queryRequest := *queryRequestPtr
	itr := q.tailQuery(r.Context(), &queryRequest)
	stream := logproto.Stream{}

	for itr.Next() {
		stream.Entries = []logproto.Entry{itr.Entry()}
		stream.Labels = itr.Labels()

		err := conn.WriteJSON(stream)
		if err != nil {
			level.Error(util.Logger).Log("Error writing to websocket", fmt.Sprintf("%v", err))
			if err := conn.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseInternalServerErr, err.Error())); err != nil {
				level.Error(util.Logger).Log("Error writing close message to websocket", fmt.Sprintf("%v", err))
			}
			break
		}
	}

	if err := itr.Error(); err != nil {
		level.Error(util.Logger).Log("Error from iterator", fmt.Sprintf("%v", err))
		if err := conn.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseInternalServerErr, err.Error())); err != nil {
			level.Error(util.Logger).Log("Error writing close message to websocket", fmt.Sprintf("%v", err))
		}
	}
}

```

```go
func (t *tailIterator) Next() bool {
	var err error
	var now time.Time

	for t.entryIterator == nil || !t.entryIterator.Next() {
		t.queryRequest.End, now = t.queryRequest.Start.Add(tailIteratorIncrement), time.Now()
		if t.queryRequest.End.After(now.Add(-delayQuerying)) {
			time.Sleep(t.queryRequest.End.Sub(now.Add(-delayQuerying)))
		}

		t.entryIterator, err = t.query()
		if err != nil {
			t.err = err
			return false
		}

		// We store the through time such that if we don't see any entries, we will
		// still make forward progress. This is overwritten by any entries we might
		// see to ensure pagination works.
		t.queryRequest.Start = t.queryRequest.End
	}

	return true
}
```

#### 堆建立

```go
func (t *tailIterator) query() (iter.EntryIterator, error) {
	ingesterIterators, err := t.querier.queryIngesters(t.ctx, t.queryRequest)
	if err != nil {
		return nil, err
	}

	chunkStoreIterators, err := t.querier.queryStore(t.ctx, t.queryRequest)
	if err != nil {
		return nil, err
	}

	iterators := append(chunkStoreIterators, ingesterIterators...)
	return iter.NewHeapIterator(iterators, t.queryRequest.Direction), nil
}
```

#### t.querier.queryIngesters ingester查找

```go
func (q *Querier) queryIngesters(ctx context.Context, req *logproto.QueryRequest) ([]iter.EntryIterator, error) {
	clients, err := q.forAllIngesters(func(client logproto.QuerierClient) (interface{}, error) {
		return client.Query(ctx, req)
	})
	if err != nil {
		return nil, err
	}

	iterators := make([]iter.EntryIterator, len(clients))
	for i := range clients {
		iterators[i] = iter.NewQueryClientIterator(clients[i].(logproto.Querier_QueryClient), req.Direction)
	}
	return iterators, nil
}
```

#### t.querier.queryStore chunk Store查找

```go
func (q Querier) queryStore(ctx context.Context, req *logproto.QueryRequest) ([]iter.EntryIterator, error) {
	matchers, err := parser.Matchers(req.Query)
	if err != nil {
		return nil, err
	}

	nameLabelMatcher, err := labels.NewMatcher(labels.MatchEqual, labels.MetricName, "logs")
	if err != nil {
		return nil, err
	}

	matchers = append(matchers, nameLabelMatcher)
	from, through := model.TimeFromUnixNano(req.Start.UnixNano()), model.TimeFromUnixNano(req.End.UnixNano())
	chks, fetchers, err := q.store.GetChunkRefs(ctx, from, through, matchers...)
	if err != nil {
		return nil, err
	}

	for i := range chks {
		chks[i] = filterChunksByTime(from, through, chks[i])
	}

	chksBySeries := partitionBySeriesChunks(chks, fetchers)
	// Make sure the initial chunks are loaded. This is not one chunk
	// per series, but rather a chunk per non-overlapping iterator.
	if err := loadFirstChunks(ctx, chksBySeries); err != nil {
		return nil, err
	}

	// Now that we have the first chunk for each series loaded,
	// we can proceed to filter the series that don't match.
	chksBySeries = filterSeriesByMatchers(chksBySeries, matchers)

	return buildIterators(ctx, req, chksBySeries)
}
```

#### 调用各个chunk store 的GetChunk方法

先查内存，后才持久层

```go
// FetchChunks fetchers a set of chunks from cache and store.
func (c *Fetcher) FetchChunks(ctx context.Context, chunks []Chunk, keys []string) ([]Chunk, error) {
	log, ctx := spanlogger.New(ctx, "ChunkStore.fetchChunks")
	defer log.Span.Finish()

	// Now fetch the actual chunk data from Memcache / S3
	cacheHits, cacheBufs, _ := c.cache.Fetch(ctx, keys)

	fromCache, missing, err := c.processCacheResponse(chunks, cacheHits, cacheBufs)
	if err != nil {
		level.Warn(log).Log("msg", "error fetching from cache", "err", err)
	}

	var fromStorage []Chunk
	if len(missing) > 0 {
		fromStorage, err = c.storage.GetChunks(ctx, missing)
	}

	// Always cache any chunks we did get
	if cacheErr := c.writeBackCache(ctx, fromStorage); cacheErr != nil {
		level.Warn(log).Log("msg", "could not store chunks in chunk cache", "err", cacheErr)
	}

	if err != nil {
		return nil, promql.ErrStorage{Err: err}
	}

	allChunks := append(fromCache, fromStorage...)
	return allChunks, nil
}
```

### Query 分析

#### 入口

```go
// pkg/querier/http.go
// QueryHandler is a http.HandlerFunc for queries.
func (q *Querier) QueryHandler(w http.ResponseWriter, r *http.Request) {
  // 构建request
	request, err := httpRequestToQueryRequest(r)
	if err != nil {
		server.WriteError(w, err)
		return
	}

	level.Debug(util.Logger).Log("request", fmt.Sprintf("%+v", request))
  // 调用grpc进行查询
	result, err := q.Query(r.Context(), request)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	if err := json.NewEncoder(w).Encode(result); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
}
```

#### grpc 查询结果

```go
// pkg/querier/querier.go
// Query does the heavy lifting for an actual query.
func (q *Querier) Query(ctx context.Context, req *logproto.QueryRequest) (*logproto.QueryResponse, error) {
  // grpc Ingester内存查找
	ingesterIterators, err := q.queryIngesters(ctx, req)
	if err != nil {
		return nil, err
	}
  // grpc Ingester持久化查找
	chunkStoreIterators, err := q.queryStore(ctx, req)
	if err != nil {
		return nil, err
	}

	iterators := append(chunkStoreIterators, ingesterIterators...)
	iterator := iter.NewHeapIterator(iterators, req.Direction)
	defer helpers.LogError("closing iterator", iterator.Close)

	resp, _, err := ReadBatch(iterator, req.Limit)
	return resp, err
}
```

#### grpc Ingester内存查找

```go
// pkg/ingester/instance.go
func (i *instance) Query(req *logproto.QueryRequest, queryServer logproto.Querier_QueryServer) error {
  // 查询解析
	matchers, err := parser.Matchers(req.Query)
	if err != nil {
		return err
	}
  // 真实查询
	iterators, err := i.lookupStreams(req, matchers)
	if err != nil {
		return err
	}

	iterator := iter.NewHeapIterator(iterators, req.Direction)
	defer helpers.LogError("closing iterator", iterator.Close)
  // 正则过滤
	if req.Regex != "" {
		var err error
		iterator, err = iter.NewRegexpFilter(req.Regex, iterator)
		if err != nil {
			return err
		}
	}

	return sendBatches(iterator, queryServer, req.Limit)
}
```

#### schema 初始化

```go
// github.com/cortexproject/cortex/pkg/chunk/schema_config.go
func (cfg PeriodConfig) createSchema() Schema {
	var s schema
	switch cfg.Schema {
	case "v1":
		s = schema{cfg.hourlyBuckets, originalEntries{}}
	case "v2":
		s = schema{cfg.dailyBuckets, originalEntries{}}
	case "v3":
		s = schema{cfg.dailyBuckets, base64Entries{originalEntries{}}}
	case "v4":
		s = schema{cfg.dailyBuckets, labelNameInHashKeyEntries{}}
	case "v5":
		s = schema{cfg.dailyBuckets, v5Entries{}}
	case "v6":
		s = schema{cfg.dailyBuckets, v6Entries{}}
	case "v9":
		s = schema{cfg.dailyBuckets, v9Entries{}}
	case "v10":
		rowShards := uint32(16)
		if cfg.RowShards > 0 {
			rowShards = cfg.RowShards
		}

		s = schema{cfg.dailyBuckets, v10Entries{
			rowShards: rowShards,
		}}
	}
	return s
}

```

#### grpc chunck持久化查找

1. 基于label标签，从index 中获取到seriesId，基于seriesId 获取到chunkId，然后通过chunkId 获取到对应的chunk 数据详情
2. 对chunk 构建堆

```go
// pkg/querier/store.go
func (q Querier) queryStore(ctx context.Context, req *logproto.QueryRequest) ([]iter.EntryIterator, error) {
  // []*labels.Matcher
	matchers, err := parser.Matchers(req.Query)
	if err != nil {
		return nil, err
	}
  // Type:labels.MatchEqual Name: __name__ Value: logs
  // Type:labels.MatchEqual Name: target Value: 192.168.153.75
	nameLabelMatcher, err := labels.NewMatcher(labels.MatchEqual, labels.MetricName, "logs")
	if err != nil {
		return nil, err
	}

	matchers = append(matchers, nameLabelMatcher)
	from, through := model.TimeFromUnixNano(req.Start.UnixNano()), model.TimeFromUnixNano(req.End.UnixNano())
  // 获取chuks 和 fetchers
	chks, fetchers, err := q.store.GetChunkRefs(ctx, from, through, matchers...)
	if err != nil {
		return nil, err
	}

	for i := range chks {
		chks[i] = filterChunksByTime(from, through, chks[i])
	}

	chksBySeries := partitionBySeriesChunks(chks, fetchers)
	// Make sure the initial chunks are loaded. This is not one chunk
	// per series, but rather a chunk per non-overlapping iterator.
  // 获取chunk
	if err := loadFirstChunks(ctx, chksBySeries); err != nil {
		return nil, err
	}

	// Now that we have the first chunk for each series loaded,
	// we can proceed to filter the series that don't match.
	chksBySeries = filterSeriesByMatchers(chksBySeries, matchers)

	return buildIterators(ctx, req, chksBySeries)
}
```



```go
// github.com/cortexproject/cortex/pkg/chunk/composite_store.go
func (c compositeStore) GetChunkRefs(ctx context.Context, from, through model.Time, matchers ...*labels.Matcher) ([][]Chunk, []*Fetcher, error) {
	chunkIDs := [][]Chunk{}
	fetchers := []*Fetcher{}
	err := c.forStores(from, through, func(from, through model.Time, store Store) error {
		ids, fetcher, err := store.GetChunkRefs(ctx, from, through, matchers...)
		if err != nil {
			return err
		}

		chunkIDs = append(chunkIDs, ids...)
		fetchers = append(fetchers, fetcher...)
		return nil
	})
	return chunkIDs, fetchers, err
}
```

##### 核心模块

```go
// github.com/cortexproject/cortex/pkg/chunk/series_store.go
func (c *seriesStore) GetChunkRefs(ctx context.Context, from, through model.Time, allMatchers ...*labels.Matcher) ([][]Chunk, []*Fetcher, error) {
	log, ctx := spanlogger.New(ctx, "SeriesStore.GetChunkRefs")
	defer log.Span.Finish()

	// Validate the query is within reasonable bounds.
  // metricName = logs
  // matchers 从allMatchers 去掉 nameMatcher 即 __name__
	metricName, matchers, shortcut, err := c.validateQuery(ctx, from, &through, allMatchers)
	if err != nil {
		return nil, nil, err
	} else if shortcut {
		return nil, nil, nil
	}

	level.Debug(log).Log("metric", metricName)

	// Fetch the series IDs from the index, based on non-empty matchers from
	// the query.
  // 去掉空值
	_, matchers = util.SplitFiltersAndMatchers(matchers)
  // 11111 读索引数据, 基于索引得到seriesIDs
	seriesIDs, err := c.lookupSeriesByMetricNameMatchers(ctx, from, through, metricName, matchers)
	if err != nil {
		return nil, nil, err
	}
	level.Debug(log).Log("series-ids", len(seriesIDs))

	// Lookup the series in the index to get the chunks.
  // 22222 基于seriesId 获取到chunkId
	chunkIDs, err := c.lookupChunksBySeries(ctx, from, through, seriesIDs)
	if err != nil {
		level.Error(log).Log("msg", "lookupChunksBySeries", "err", err)
		return nil, nil, err
	}
	level.Debug(log).Log("chunk-ids", len(chunkIDs))
  // 33333 基于chunkId 获取到chunk流数据
	chunks, err := c.convertChunkIDsToChunks(ctx, chunkIDs)
	if err != nil {
		level.Error(log).Log("op", "convertChunkIDsToChunks", "err", err)
		return nil, nil, err
	}

	return [][]Chunk{chunks}, []*Fetcher{c.store.Fetcher}, nil
}
```

##### 读索引数据, 基于索引得到seriesIDs

```go
// 11111 读索引数据, 基于索引得到seriesIDs
// cortexproject/cortex/pkg/chunk/series_store.go
func (c *seriesStore) lookupSeriesByMetricNameMatcher(ctx context.Context, from, through model.Time, metricName string, matcher *labels.Matcher) ([]string, error) {
	log, ctx := spanlogger.New(ctx, "SeriesStore.lookupSeriesByMetricNameMatcher", "metricName", metricName, "matcher", matcher)
	defer log.Span.Finish()

	userID, err := user.ExtractOrgID(ctx)
	if err != nil {
		return nil, err
	}

	var queries []IndexQuery
	if matcher == nil {
    // 场景1
		queries, err = c.schema.GetReadQueriesForMetric(from, through, userID, model.LabelValue(metricName))
	} else if matcher.Type != labels.MatchEqual {
    // 场景2 
		queries, err = c.schema.GetReadQueriesForMetricLabel(from, through, userID, model.LabelValue(metricName), model.LabelName(matcher.Name))
	} else {
    // 场景3 label 等于
    
    // 基于from 和 through  按天顺序读取，最大跨度是一天，精确到毫秒
    // 实际entry，基于schema 配置，默认v9
    /*
     schema :
     result = append(result, Bucket{
			from:      uint32(relativeFrom),
			through:   uint32(relativeThrough),
			tableName: cfg.tableForBucket(i * secondsInDay),
			hashKey:   fmt.Sprintf("%s:d%d", userID, i), // fake:d:18025
		})
    */
    /*
    v9:GetReadMetricLabelValueQueries
    func (v9Entries) GetReadMetricLabelValueQueries(bucket Bucket, metricName model.LabelValue, labelName model.LabelName, labelValue model.LabelValue) ([]IndexQuery, error) {
    // labelValue="192.168.153.75"
    // valueHash = []byte()
	valueHash := sha256bytes(string(labelValue))
	return []IndexQuery{
		{
			TableName:       bucket.tableName,
			// bucket.hashKey = fake:18025:18025:logs:target
			HashValue:       fmt.Sprintf("%s:%s:%s", bucket.hashKey, metricName, labelName),
			// []byte ==> string:  RangeValueStart=n5Nuug0kaHLJyF8FyXSrDh2Bk2UcZyTqzCVJr8DTJNk 
			RangeValueStart: encodeRangeKey(valueHash),
			ValueEqual:      []byte(labelValue),
		},
	}, nil
}
    */
		queries, err = c.schema.GetReadQueriesForMetricLabelValue(from, through, userID, model.LabelValue(metricName), model.LabelName(matcher.Name), model.LabelValue(matcher.Value))
	}
	if err != nil {
		return nil, err
	}
	level.Debug(log).Log("queries", len(queries))
  // 持久化查找index， 从索引数据库中查找
	entries, err := c.lookupEntriesByQueries(ctx, queries)
	if err != nil {
		return nil, err
	}
	level.Debug(log).Log("entries", len(entries))

  // seriesId
  /*
  chunkKey, labelValue, _, _, err := parseChunkTimeRangeValue(entry.RangeValue, entry.Value)
	对entries 做一层过滤刷选
  */
	ids, err := c.parseIndexEntries(ctx, entries, matcher)
	if err != nil {
		return nil, err
	}
	level.Debug(log).Log("ids", len(ids))

	return ids, nil
}
```

##### 查找持久化的index

```go

func (c *store) lookupEntriesByQueries(ctx context.Context, queries []IndexQuery) ([]IndexEntry, error) {
	var lock sync.Mutex
	var entries []IndexEntry
  // 持久化查找
	err := c.index.QueryPages(ctx, queries, func(query IndexQuery, resp ReadBatch) bool {
		iter := resp.Iterator()
		lock.Lock()
		for iter.Next() {
			entries = append(entries, IndexEntry{
				TableName:  query.TableName,
				HashValue:  query.HashValue,
				RangeValue: iter.RangeValue(),
				Value:      iter.Value(),
			})
		}
		lock.Unlock()
		return true
	})
	if err != nil {
		level.Error(util.WithContext(ctx, util.Logger)).Log("msg", "error querying storage", "err", err)
	}
	return entries, err
}
```

##### 基于seriesId 找到chunkId

```go
// 22222 基于seriesId 获取到chunkId
// cortexproject/cortex/pkg/chunk/series_store.go
func (c *seriesStore) lookupChunksBySeries(ctx context.Context, from, through model.Time, seriesIDs []string) ([]string, error) {
	log, ctx := spanlogger.New(ctx, "SeriesStore.lookupChunksBySeries")
	defer log.Span.Finish()

	userID, err := user.ExtractOrgID(ctx)
	if err != nil {
		return nil, err
	}
	level.Debug(log).Log("seriesIDs", len(seriesIDs))

	queries := make([]IndexQuery, 0, len(seriesIDs))
	for _, seriesID := range seriesIDs {
    /*
    func (v9Entries) GetChunksForSeries(bucket Bucket, seriesID []byte) ([]IndexQuery, error) {
	encodedFromBytes := encodeTime(bucket.from)
	return []IndexQuery{
		{
			TableName:       bucket.tableName,
			HashValue:       bucket.hashKey + ":" + string(seriesID),
			RangeValueStart: encodeRangeKey(encodedFromBytes),
		},
	}, nil
}
    */
		qs, err := c.schema.GetChunksForSeries(from, through, userID, []byte(seriesID))
		if err != nil {
			return nil, err
		}
		queries = append(queries, qs...)
	}
	level.Debug(log).Log("queries", len(queries))
/*
func (c *store) lookupEntriesByQueries(ctx context.Context, queries []IndexQuery) ([]IndexEntry, error) {
	var lock sync.Mutex
	var entries []IndexEntry
	err := c.index.QueryPages(ctx, queries, func(query IndexQuery, resp ReadBatch) bool {
		iter := resp.Iterator()
		lock.Lock()
		for iter.Next() {
			entries = append(entries, IndexEntry{
				TableName:  query.TableName,
				HashValue:  query.HashValue,
				RangeValue: iter.RangeValue(),
				Value:      iter.Value(),
			})
		}
		lock.Unlock()
		return true
	})
	if err != nil {
		level.Error(util.WithContext(ctx, util.Logger)).Log("msg", "error querying storage", "err", err)
	}
	return entries, err
}
*/
	entries, err := c.lookupEntriesByQueries(ctx, queries)
	if err != nil {
		return nil, err
	}
	level.Debug(log).Log("entries", len(entries))

	result, err := c.parseIndexEntries(ctx, entries, nil)
	return result, err
}
```

##### 基于chunkId 获取到chunk流数据

```go
// 33333 基于chunkId 获取到chunk流数据
// github.com/cortexproject/cortex/pkg/chunk/chunk_store.go
func (c *store) convertChunkIDsToChunks(ctx context.Context, chunkIDs []string) ([]Chunk, error) {
	userID, err := user.ExtractOrgID(ctx)
	if err != nil {
		return nil, err
	}

	chunkSet := make([]Chunk, 0, len(chunkIDs))
	for _, chunkID := range chunkIDs {
		chunk, err := ParseExternalKey(userID, chunkID)
		if err != nil {
			return nil, err
		}
		chunkSet = append(chunkSet, chunk)
	}

	return chunkSet, nil
}
```

```go
// github.com/cortexproject/cortex/pkg/chunk/chunk.go
// ParseExternalKey is used to construct a partially-populated chunk from the
// key in DynamoDB.  This chunk can then be used to calculate the key needed
// to fetch the Chunk data from Memcache/S3, and then fully populate the chunk
// with decode().
//
// Pre-checksums, the keys written to DynamoDB looked like
// `<fingerprint>:<start time>:<end time>` (aka the ID), and the key for
// memcache and S3 was `<user id>/<fingerprint>:<start time>:<end time>.
// Finger prints and times were written in base-10.
//
// Post-checksums, externals keys become the same across DynamoDB, Memcache
// and S3.  Numbers become hex encoded.  Keys look like:
// `<user id>/<fingerprint>:<start time>:<end time>:<checksum>`.
func ParseExternalKey(userID, externalKey string) (Chunk, error) {
	if !strings.Contains(externalKey, "/") {
		return parseLegacyChunkID(userID, externalKey)
	}
	chunk, err := parseNewExternalKey(externalKey)
	if err != nil {
		return Chunk{}, err
	}
	if chunk.UserID != userID {
		return Chunk{}, errors.WithStack(ErrWrongMetadata)
	}
	return chunk, nil
}
```



```go

// FetchChunks fetchers a set of chunks from cache and store.
func (c *Fetcher) FetchChunks(ctx context.Context, chunks []Chunk, keys []string) ([]Chunk, error) {
	log, ctx := spanlogger.New(ctx, "ChunkStore.fetchChunks")
	defer log.Span.Finish()

	// Now fetch the actual chunk data from Memcache / S3
  // 内存
	cacheHits, cacheBufs, _ := c.cache.Fetch(ctx, keys)
 
	fromCache, missing, err := c.processCacheResponse(chunks, cacheHits, cacheBufs)
	if err != nil {
		level.Warn(log).Log("msg", "error fetching from cache", "err", err)
	}

	var fromStorage []Chunk
	if len(missing) > 0 {
    // 持久化获取
		fromStorage, err = c.storage.GetChunks(ctx, missing)
	}

	// Always cache any chunks we did get
	if cacheErr := c.writeBackCache(ctx, fromStorage); cacheErr != nil {
		level.Warn(log).Log("msg", "could not store chunks in chunk cache", "err", cacheErr)
	}

	if err != nil {
		return nil, promql.ErrStorage{Err: err}
	}

	allChunks := append(fromCache, fromStorage...)
	return allChunks, nil
}
```

#### s3 获取chunk

```go

// GetParallelChunks fetches chunks in parallel (up to maxParallel).
func GetParallelChunks(ctx context.Context, chunks []chunk.Chunk, f func(context.Context, *chunk.DecodeContext, chunk.Chunk) (chunk.Chunk, error)) ([]chunk.Chunk, error) {
	sp, ctx := ot.StartSpanFromContext(ctx, "GetParallelChunks")
	defer sp.Finish()
	sp.LogFields(otlog.Int("chunks requested", len(chunks)))

	queuedChunks := make(chan chunk.Chunk)

	go func() {
		for _, c := range chunks {
			queuedChunks <- c
		}
		close(queuedChunks)
	}()

	processedChunks := make(chan chunk.Chunk)
	errors := make(chan error)

	for i := 0; i < min(maxParallel, len(chunks)); i++ {
		go func() {
			decodeContext := chunk.NewDecodeContext()
			for c := range queuedChunks {
				c, err := f(ctx, decodeContext, c)
				if err != nil {
					errors <- err
				} else {
					processedChunks <- c
				}
			}
		}()
	}

	var result = make([]chunk.Chunk, 0, len(chunks))
	var lastErr error
	for i := 0; i < len(chunks); i++ {
		select {
		case chunk := <-processedChunks:
			result = append(result, chunk)
		case err := <-errors:
			lastErr = err
		}
	}

	sp.LogFields(otlog.Int("chunks fetched", len(result)))
	if lastErr != nil {
		sp.LogFields(otlog.Error(lastErr))
	}

	// Return any chunks we did receive: a partial result may be useful
	return result, lastErr
}
```

```go
// s3 从持久化中获取chunk
func (a s3ObjectClient) getChunk(ctx context.Context, decodeContext *chunk.DecodeContext, c chunk.Chunk) (chunk.Chunk, error) {
	var resp *s3.GetObjectOutput
	err := instrument.CollectedRequest(ctx, "S3.GetObject", s3RequestDuration, instrument.ErrorCode, func(ctx context.Context) error {
		var err error
    // 真实获取s3
		resp, err = a.S3.GetObjectWithContext(ctx, &s3.GetObjectInput{
			Bucket: aws.String(a.bucketName),
			Key:    aws.String(c.ExternalKey()),
		})
		return err
	})
	if err != nil {
		return chunk.Chunk{}, err
	}
	defer resp.Body.Close()
	buf, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return chunk.Chunk{}, err
	}
	if err := c.Decode(decodeContext, buf); err != nil {
		return chunk.Chunk{}, err
	}
	return c, nil
}
```

#### chunk 定义

```go

// Chunk contains encoded timeseries data
type Chunk struct {
	// These two fields will be missing from older chunks (as will the hash).
	// On fetch we will initialise these fields from the DynamoDB key.
	Fingerprint model.Fingerprint `json:"fingerprint"`
	UserID      string            `json:"userID"`

	// These fields will be in all chunks, including old ones.
	From    model.Time   `json:"from"`
	Through model.Time   `json:"through"`
	Metric  model.Metric `json:"metric"`

	// The hash is not written to the external storage either.  We use
	// crc32, Castagnoli table.  See http://www.evanjones.ca/crc32c.html.
	// For old chunks, ChecksumSet will be false.
	ChecksumSet bool   `json:"-"`
	Checksum    uint32 `json:"-"`

	// We never use Delta encoding (the zero value), so if this entry is
	// missing, we default to DoubleDelta.
	Encoding prom_chunk.Encoding `json:"encoding"`
	Data     prom_chunk.Chunk    `json:"-"`

	// This flag is used for very old chunks, where the metadata is read out
	// of the index.
	metadataInIndex bool

	// The encoded version of the chunk, held so we don't need to re-encode it
	encoded []byte
}
```

#### 索引表

```go
// 索引表名
// cortexproject/cortex/pkg/chunk/schema_config.go
func (cfg *PeriodConfig) tableForBucket(bucketStart int64) string {
	if cfg.IndexTables.Period == 0 {
		return cfg.IndexTables.Prefix
	}
	// TODO remove reference to time package here
	return cfg.IndexTables.Prefix + strconv.Itoa(int(bucketStart/int64(cfg.IndexTables.Period/time.Second)))
}

// 反向推到索引创建时间	
bucketStart := int64(168 * time.Hour/time.Second) * i
```

