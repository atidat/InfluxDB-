**engine Interface**



**engine Type**

Engine：存储引擎具体实现，存储了配置

```go
type Engine struct {
    config			Config
    path			string
    mu				sync.RWMutex
    closing			chan struct{}
    tsdbStore		*tsdb.Store
    metaClient		MetaClient
    pointsWriter	interface {
        WriterPoints(ctx context.Context, database, relationPolicy string, consistencyLevel models.ConsistencyLevel, user meta.User, points []models.Point) error
        Close() error
    }
    // 数据保留策略的强制执行服务
    retentionService				*retention.Service
    // Shard预创建服务
    precreatorService				*precreator.Service
    writePointsValidationEnabled	bool
    logger							*zap.Logger
    metricsDisabled					bool
    
}
```

func (e *Engine) WithLogger(log *zap.Logger)：TODO

func(e *Engine)PrometheusCollectors() []prometheus.Collector：TODO

func (e *Engine) Open(ctx context.Context) error：打开调用链服务、tsdb存储服务、数据保留服务和shard预创建服务
```go
func (e *Engine) Open(ctx context.Context) (err error) {
	e.mu.Lock()			// 加写锁
	defer e.mu.Unlock()

	if e.closing != nil { return nil // Already open }

	span, _ := tracing.StartSpanFromContext(ctx) // 调用链服务
	defer span.Finish()

	if err := e.tsdbStore.Open(ctx); err != nil { return err }
	if err := e.retentionService.Open(ctx); err != nil { return err }
	if err := e.precreatorService.Open(ctx); err != nil { return err }
	e.closing = make(chan struct{})
	return nil
}
```

func (e *Engine) Close() error：关闭调用链服务、tsdb存储服务、数据保留服务、shard预创建服务和<u>point写服务</u>
```go
func (e *Engine) Close() error {
	e.mu.RLock()	// 加读锁
	if e.closing == nil {
		e.mu.RUnlock()
		// Unusual if an engine is closed more than once, so note it.
		e.logger.Info("Close() called on already-closed engine")
		return nil // Already closed
	}

	close(e.closing)
	e.mu.RUnlock()

	e.mu.Lock()		// 加写锁
	defer e.mu.Unlock()
	e.closing = nil

	var retErr error
	if err := e.precreatorService.Close(); err != nil {
		retErr = multierr.Append(retErr, fmt.Errorf("error closing shard precreator service: %w", err))
	}

	if err := e.retentionService.Close(); err != nil {
		retErr = multierr.Append(retErr, fmt.Errorf("error closing retention service: %w", err))
	}

	if err := e.tsdbStore.Close(); err != nil {
		retErr = multierr.Append(retErr, fmt.Errorf("error closing TSDB store: %w", err))
	}

	if err := e.pointsWriter.Close(); err != nil {
		retErr = multierr.Append(retErr, fmt.Errorf("error closing points writer: %w", err))
	}
	return retErr
}
```

func (e *Engine) WritePoints(ctx context.Context, orgID platform.ID, bucketID platform.ID, points []models.Point) error：




**engine API**

WithMetaClient(c MetaClient) func(*Engine)：设置引擎的MetaClient选项

WithMetricsDisabled(m bool) func(*Engine)：禁能引擎的Metrics选项

注：上述通过选项模式设置存储引擎的配置

