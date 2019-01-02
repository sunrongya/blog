# go-kit 设计精髓
[go-kit](https://github.com/go-kit/kit) 为 Golang 微服务开发者提供一系列工具包，使开发者可以更专注业务逻辑。

## 分层结构
- Transport
- Endpoint
- Service

## Transport
封装网络传输层，如：HTTP,GRPC,NATS,AMQP 等。

## Service
用户自定义业务服务，专注业务逻辑实现，完全解耦合。如：
```go
// StringService provides operations on strings.
type StringService interface {
    Uppercase(string) (string, error)
    Count(string) int
}
```

## Endpoint
> 由于 [go-kit](https://github.com/go-kit/kit) 的核心是 Endpoint, 就放在最后。

```go
// Endpoint is the fundamental building block of servers and clients.
// It represents a single RPC method.
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```

```go
// Middleware is a chainable behavior modifier for endpoints.
type Middleware func(Endpoint) Endpoint

// Chain is a helper function for composing middlewares. Requests will
// traverse them in the order they're declared. That is, the first middleware
// is treated as the outermost middleware.
func Chain(outer Middleware, others ...Middleware) Middleware {
    return func(next Endpoint) Endpoint {
        for i := len(others) - 1; i >= 0; i-- { // reverse
            next = others[i](next)
        }
        return outer(next)
    }
}
```

[go-kit](https://github.com/go-kit/kit) 通过抽象类型 Endpoint 把业务服务及中间件组合起来。抽象类型 Endpoint 定义为：接收上下文 (Context) 及请求参数 (request), 并做出响应 (response, err) 的一类抽象（学过函数式编程，如 Haskell, 特别是范畴论的同学对这里应该比较有亲切感）。

> 具体通过 Endpoint 抽象怎么组合后续会分析（希望没忘记）。

## 其它
由于 [go-kit](https://github.com/go-kit/kit) 开发过程有很多重复操作，可以通过代码生成工具生成框架代码，使开发更专注业务逻辑。
- 代码生成工具 [Truss](https://github.com/gitwak/truss)

