---
title: nsqd源码分析之nsqd 进程的数据结构 (基于 1.2.0 版本)
tag: [golang,nsq]
---

以 [1.2.0](https://github.com/nsqio/nsq/releases/tag/v1.2.0) 版本为准.

## 涉及的数据结构
- NSQD
- Topic
- Channel
- Message
- inFlightPqueue
- PriorityQueue
- diskQueue
- dummyBackendQueue

以上数据结构的关系如下图:
![nsqd 数据结构关系图](/image/2020/04/20/nsqd数据结构.png)

## NSQD

[/nsqd/nsqd.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/nsqd.go#L44)
``` go
type NSQD struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms
	clientIDSequence int64

	sync.RWMutex

	opts atomic.Value

	dl        *dirlock.DirLock
	isLoading int32
	errValue  atomic.Value
	startTime time.Time

	// 这里有 Topic
	// one
	topicMap map[string]*Topic

	clientLock sync.RWMutex
	clients    map[int64]Client

	lookupPeers atomic.Value

	tcpServer     *tcpServer
	tcpListener   net.Listener
	httpListener  net.Listener
	httpsListener net.Listener
	tlsConfig     *tls.Config

	poolSize int

	notifyChan           chan interface{}
	optsNotificationChan chan struct{}
	exitChan             chan int
	waitGroup            util.WaitGroupWrapper

	ci *clusterinfo.ClusterInfo
}
```

## Topic

[/nsqd/topic.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/topic.go#L17)
``` go
type Topic struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms
	messageCount uint64
	messageBytes uint64

	sync.RWMutex

	name              string
	channelMap        map[string]*Channel
	backend           BackendQueue
	memoryMsgChan     chan *Message
	startChan         chan int
	exitChan          chan int
	channelUpdateChan chan int
	waitGroup         util.WaitGroupWrapper
	exitFlag          int32
	idFactory         *guidFactory

	ephemeral      bool
	deleteCallback func(*Topic)
	deleter        sync.Once

	paused    int32
	pauseChan chan int

	ctx *context
}
```

## Channel
[/nsqd/channel.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/channel.go#L36)
``` go
// Channel represents the concrete type for a NSQ channel (and also
// implements the Queue interface)
//
// There can be multiple channels per topic, each with there own unique set
// of subscribers (clients).
//
// Channels maintain all client and message metadata, orchestrating in-flight
// messages, timeouts, requeuing, etc.
type Channel struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms
	requeueCount uint64
	messageCount uint64
	timeoutCount uint64

	sync.RWMutex

	topicName string
	name      string
	ctx       *context

	backend BackendQueue

	memoryMsgChan chan *Message
	exitFlag      int32
	exitMutex     sync.RWMutex

	// state tracking
	clients        map[int64]Consumer
	paused         int32
	ephemeral      bool
	deleteCallback func(*Channel)
	deleter        sync.Once

	// Stats tracking
	e2eProcessingLatencyStream *quantile.Quantile

	// TODO: these can be DRYd up
	deferredMessages map[MessageID]*pqueue.Item
	deferredPQ       pqueue.PriorityQueue
	deferredMutex    sync.Mutex
	inFlightMessages map[MessageID]*Message
	inFlightPQ       inFlightPqueue
	inFlightMutex    sync.Mutex
}
```

## Message

[/nsqd/message.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/message.go#L18)
``` go
type Message struct {
	ID        MessageID
	Body      []byte
	Timestamp int64
	Attempts  uint16

	// for in-flight handling
	deliveryTS time.Time
	clientID   int64
	pri        int64
	index      int
	deferred   time.Duration
}
```

## Queue

#### inFlightPqueue
[/nsqd/in_flight_pqueue.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/in_flight_pqueue.go#L3)

``` go
type inFlightPqueue []*Message
```

#### PriorityQueue
[/internal/pqueue/pqueue.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/internal/pqueue/pqueue.go#L7)
``` go
type Item struct {
	Value    interface{}
	Priority int64
	Index    int
}

// this is a priority queue as implemented by a min heap
// ie. the 0th element is the *lowest* value
type PriorityQueue []*Item
```

#### BackendQueue

[/nsqd/backend_queue.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/backend_queue.go#L5)
``` go
// BackendQueue represents the behavior for the secondary message
// storage system
type BackendQueue interface {
	Put([]byte) error
	ReadChan() <-chan []byte // this is expected to be an *unbuffered* channel
	Close() error
	Delete() error
	Depth() int64
	Empty() error
}
```

##### diskQueue
[/diskqueue.go](https://github.com/nsqio/go-diskqueue/blob/74cfbc9de8399a7fa86d676b908b002120826fc0/diskqueue.go#L58)
``` go
// diskQueue implements a filesystem backed FIFO queue
type diskQueue struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms

	// run-time state (also persisted to disk)
	readPos      int64
	writePos     int64
	readFileNum  int64
	writeFileNum int64
	depth        int64

	sync.RWMutex

	// instantiation time metadata
	name            string
	dataPath        string
	maxBytesPerFile int64 // currently this cannot change once created
	minMsgSize      int32
	maxMsgSize      int32
	syncEvery       int64         // number of writes per fsync
	syncTimeout     time.Duration // duration of time per fsync
	exitFlag        int32
	needSync        bool

	// keeps track of the position where we have read
	// (but not yet sent over readChan)
	nextReadPos     int64
	nextReadFileNum int64

	readFile  *os.File
	writeFile *os.File
	reader    *bufio.Reader
	writeBuf  bytes.Buffer

	// exposed via ReadChan()
	readChan chan []byte

	// internal channels
	depthChan         chan int64
	writeChan         chan []byte
	writeResponseChan chan error
	emptyChan         chan int
	emptyResponseChan chan error
	exitChan          chan int
	exitSyncChan      chan int

	logf AppLogFunc
}
```

##### dummyBackendQueue
[/nsqd/dummy_backend_queue.go](https://github.com/nsqio/nsq/blob/e7d872a9a8b562695be39b519eadce00771874eb/nsqd/dummy_backend_queue.go#L3)
``` go
type dummyBackendQueue struct {
	readChan chan []byte
}
```