---
title: nsqd的启动过程分析(基于 1.2.0 版本)
tag: [golang,nsq]
---

以 [1.2.0](https://github.com/nsqio/nsq/releases/tag/v1.2.0) 版本为准.

[/apps/nsqd/main.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/apps/nsqd/main.go#L41) 中的 `Start` 方法即是 nsqd daemon 的启动过程.

``` golang
func (p *program) Start() error {
  opts := nsqd.NewOptions()
  
// ...
// ... 
  
	nsqd, err := nsqd.New(opts)
	if err != nil {
		logFatal("failed to instantiate nsqd - %s", err)
	}
	p.nsqd = nsqd

	err = p.nsqd.LoadMetadata()
	if err != nil {
		logFatal("failed to load metadata - %s", err)
	}
	err = p.nsqd.PersistMetadata()
	if err != nil {
		logFatal("failed to persist metadata - %s", err)
	}

	go func() {
		err := p.nsqd.Main()
		if err != nil {
			p.Stop()
			os.Exit(1)
		}f
	}()

	return nil
}
```

在执行到 `p.nsqd.Main` 之前, nsqd 先调用了 `LoadMetadata` 和 `PersistMetadata` 方法, 以恢复上一次 nsqd 运行时保存的运行数据.

下面来看看 [/nsqd/nsqd.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/nsqd.go#L244) 中 `Main` 方法里的具体过程.

``` go
func (n *NSQD) Main() error {
	ctx := &context{n}

	exitCh := make(chan error)
	var once sync.Once
	exitFunc := func(err error) {
		once.Do(func() {
			if err != nil {
				n.logf(LOG_FATAL, "%s", err)
			}
			exitCh <- err
		})
	}

	n.tcpServer.ctx = ctx
	n.waitGroup.Wrap(func() {
		exitFunc(protocol.TCPServer(n.tcpListener, n.tcpServer, n.logf))
	})

	httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
	n.waitGroup.Wrap(func() {
		exitFunc(http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf))
	})

	if n.tlsConfig != nil && n.getOpts().HTTPSAddress != "" {
		httpsServer := newHTTPServer(ctx, true, true)
		n.waitGroup.Wrap(func() {
			exitFunc(http_api.Serve(n.httpsListener, httpsServer, "HTTPS", n.logf))
		})
	}

	n.waitGroup.Wrap(n.queueScanLoop)
	n.waitGroup.Wrap(n.lookupLoop)
	if n.getOpts().StatsdAddress != "" {
		n.waitGroup.Wrap(n.statsdLoop)
	}

	err := <-exitCh
	return err
}
```

其中nsqd 创建了 `tcpServer` 和 `httpServer`. `tcpServer` 负责监听客户端的请求, `httpServer` 负责对外提供 [api 服务](https://nsq.io/components/nsqd.html#http-api). 如果 nsqd 启动时有配置 https, 也会创建 `httpsServer`.

另外还创建了 `queueScanLoop` 和 `lookupLoop` goroutine. `queueScanLoop` 是 nsqd 内部处理消息的定时任务, `lookupLoop` 是向 `nsqlookupd` 同步节点信息的定时任务.

如果 nsqd 启动时还配置了 **-statsd-address** 参数, 会创建 `statsLoop` 的 goroutine. 负责定时推送这个 nsqd 进程的内存信息.

以上这些服务都是以新的 goroutine 创建的, 这里采用了sync.WaitGroup这个同步工具. 这里 nsqd 对 WaitGroup 进行了一层包装.代码在
[/internal/util/wait_group_wrapper.go](https://github.com/nsqio/nsq/blob/e7d872a9a8/internal/util/wait_group_wrapper.go#L7).

``` go
type WaitGroupWrapper struct {
	sync.WaitGroup
}

func (w *WaitGroupWrapper) Wrap(cb func()) {
	w.Add(1)
	go func() {
		cb()
		w.Done()
	}()
}
```


