# 服务发现 (Service Discovery)
通过源码解析 [go-kit](https://github.com/go-kit/kit) 负载均衡流程。

## 负载均衡
- 定义接口 Balancer
- 随机及循环负载策略
- 重试

#### 负载接口 [balancer.go](https://github.com/go-kit/kit/blob/master/sd/lb/balancer.go)
```go
// Balancer yields endpoints according to some heuristic.
type Balancer interface {
    Endpoint() (endpoint.Endpoint, error)
}
```

#### 负载策略（随机及循环）
随机 [random.go](https://github.com/go-kit/kit/blob/master/sd/lb/random.go)
```go
type random struct {
    s sd.Endpointer
    r *rand.Rand
}

func (r *random) Endpoint() (endpoint.Endpoint, error) {
    endpoints, err := r.s.Endpoints()
    if err != nil {
        return nil, err
    }
    if len(endpoints) <= 0 {
        return nil, ErrNoEndpoints
    }
    return endpoints[r.r.Intn(len(endpoints))], nil
}
```

循环 [round_robin.go](https://github.com/go-kit/kit/blob/master/sd/lb/round_robin.go)
```go
type roundRobin struct {
    s sd.Endpointer
    c uint64
}

func (rr *roundRobin) Endpoint() (endpoint.Endpoint, error) {
    endpoints, err := rr.s.Endpoints()
    if err != nil {
        return nil, err
    }
    if len(endpoints) <= 0 {
        return nil, ErrNoEndpoints
    }
    old := atomic.AddUint64(&rr.c, 1) - 1
    idx := old % uint64(len(endpoints))
    return endpoints[idx], nil
}
```
#### 重试 
[retry.go](https://github.com/go-kit/kit/blob/master/sd/lb/retry.go)
```go
// Retry wraps a service load balancer and returns an endpoint oriented load
// balancer for the specified service method. Requests to the endpoint will be
// automatically load balanced via the load balancer. Requests that return
// errors will be retried until they succeed, up to max times, or until the
// timeout is elapsed, whichever comes first.
func Retry(max int, timeout time.Duration, b Balancer) endpoint.Endpoint {
    return RetryWithCallback(timeout, b, maxRetries(max))
}
```

#### lb 包小结
从上面的负载策略（随机及循环）实现中不难发现它们都通过 sd.Endpointer 获取 []endpoint.Endpoint.
```go
type Endpointer interface {
    Endpoints() ([]endpoint.Endpoint, error)
}
```

#### Endpointer 实现
- 提供 Instancer 及 Factory 创建 Endpointer（后面会介绍 Instancer 及 Factory 的用途）。
- src.Register(se.ch) 注册进 Instancer, Instancer 通过 chan Event 通知服务状态。
- se.receive() 不断监听 chan Event, 并更新服务状态。

> 服务状态流：Instancer -> chan Event -> Endpointer -> update endpointCache

```go
// NewEndpointer creates an Endpointer that subscribes to updates from Instancer src
// and uses factory f to create Endpoints. If src notifies of an error, the Endpointer
// keeps returning previously created Endpoints assuming they are still good, unless
// this behavior is disabled via InvalidateOnError option.
func NewEndpointer(src Instancer, f Factory, logger log.Logger, options ...EndpointerOption) *DefaultEndpointer {
    opts := endpointerOptions{}
    for _, opt := range options {
        opt(&opts)
    }
    se := &DefaultEndpointer{
        cache:     newEndpointCache(f, logger, opts),
        instancer: src,
        ch:        make(chan Event),
    }
    go se.receive()
    src.Register(se.ch)
    return se
}

// DefaultEndpointer implements an Endpointer interface.
// When created with NewEndpointer function, it automatically registers
// as a subscriber to events from the Instances and maintains a list
// of active Endpoints.
type DefaultEndpointer struct {
    cache     *endpointCache
    instancer Instancer
    ch        chan Event
}

func (de *DefaultEndpointer) receive() {
    for event := range de.ch {
        de.cache.Update(event)
    }
}

// Close deregisters DefaultEndpointer from the Instancer and stops the internal go-routine.
func (de *DefaultEndpointer) Close() {
    de.instancer.Deregister(de.ch)
    close(de.ch)
}

// Endpoints implements Endpointer.
func (de *DefaultEndpointer) Endpoints() ([]endpoint.Endpoint, error) {
    return de.cache.Endpoints()
}
```

#### Instancer

```go
type Event struct {
    Instances []string
    Err       error
}

// Instancer listens to a service discovery system and notifies registered
// observers of changes in the resource instances. Every event sent to the channels
// contains a complete set of instances known to the Instancer. That complete set is
// sent immediately upon registering the channel, and on any future updates from
// discovery system.
type Instancer interface {
    Register(chan<- Event)
    Deregister(chan<- Event)
    Stop()
}

```

服务发现实现 Instancer 接口，通过 chan<- Event 更新状态，如 Consul: [instancer.go](https://github.com/go-kit/kit/blob/master/sd/consul/instancer.go)

```go
// Instancer yields instances for a service in Consul.
type Instancer struct {
    cache       *instance.Cache
    client      Client
    logger      log.Logger
    service     string
    tags        []string
    passingOnly bool
    quitc       chan struct{}
}

// Register implements Instancer.
func (s *Instancer) Register(ch chan<- sd.Event) {
    s.cache.Register(ch)
}

// Deregister implements Instancer.
func (s *Instancer) Deregister(ch chan<- sd.Event) {
    s.cache.Deregister(ch)
}
```

#### Factory
具体怎么用可以参考下面的例子
```go
// Factory is a function that converts an instance string (e.g. host:port) to a
// specific endpoint. Instances that provide multiple endpoints require multiple
// factories. A factory also returns an io.Closer that's invoked when the
// instance goes away and needs to be cleaned up. Factories may return nil
// closers.
//
// Users are expected to provide their own factory functions that assume
// specific transports, or can deduce transports by parsing the instance string.
type Factory func(instance string) (endpoint.Endpoint, io.Closer, error)
```

## 实例
参考 [examples/apigateway](https://github.com/go-kit/kit/blob/master/examples/apigateway/main.go) （这里只留下对分析有帮助的代码）

```go
func main() {...
    // Service discovery domain. In this example we use Consul.
    var client consulsd.Client
    {
        client = consulsd.NewClient(consulClient)
    }
    r := mux.NewRouter()

    {
        var (
            tags        = []string{}
            passingOnly = true
            endpoints   = addendpoint.Set{}
            instancer   = consulsd.NewInstancer(client, logger, "addsvc", tags, passingOnly)
        )
        {
            factory := addsvcFactory(addendpoint.MakeSumEndpoint, ...)
            endpointer := sd.NewEndpointer(instancer, factory, logger)
            balancer := lb.NewRoundRobin(endpointer)
            retry := lb.Retry(*retryMax, *retryTimeout, balancer)
            endpoints.SumEndpoint = retry
        }

        r.PathPrefix("/addsvc").Handler(http.StripPrefix("/addsvc", addtransport.NewHTTPHandler(endpoints, tracer, zipkinTracer, logger)))
    }
}

// 这里就是 Factory 的作用（服务状态变更重新获取 (endpoint,conn), (endpoint,conn) 相同生命周期）
func addsvcFactory(makeEndpoint func(addservice.Service) endpoint.Endpoint, ...) sd.Factory {
    return func(instance string) (endpoint.Endpoint, io.Closer, error) {
        conn, err := grpc.Dial(instance, grpc.WithInsecure())
        if err != nil {
            return nil, nil, err
        }
        service := addtransport.NewGRPCClient(conn, ...)
        endpoint := makeEndpoint(service)
        return endpoint, conn, nil
    }
}
```