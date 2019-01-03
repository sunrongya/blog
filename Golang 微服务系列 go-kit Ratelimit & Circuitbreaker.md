# go-kit Ratelimit & Circuitbreaker
使用 [go-kit](https://github.com/go-kit/kit) 实现微服务限速、限流、熔断机制。主要是通过封装第三方中间件来达到组合目的 (endpoint.Middleware).

## Ratelimit 源码
```go
func NewErroringLimiter(limit Allower) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			if !limit.Allow() {
				return nil, ErrLimited
			}
			return next(ctx, request)
		}
	}
}
```

```go
func NewDelayingLimiter(limit Waiter) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			if err := limit.Wait(ctx); err != nil {
				return nil, err
			}
			return next(ctx, request)
		}
	}
}
```

## 各个 Circuitbreaker 源码
### [Gobreaker](github.com/sony/gobreaker) 封装
```go
func Gobreaker(cb *gobreaker.CircuitBreaker) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			return cb.Execute(func() (interface{}, error) { return next(ctx, request) })
		}
	}
}
```

### [Hystrix](github.com/afex/hystrix-go/hystrix) 封装
```go
func Hystrix(commandName string) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (response interface{}, err error) {
			var resp interface{}
			if err := hystrix.Do(commandName, func() (err error) {
				resp, err = next(ctx, request)
				return err
			}, nil); err != nil {
				return nil, err
			}
			return resp, nil
		}
	}
}
```

### [HandyBreaker](github.com/streadway/handy/breaker) 封装
```go
func HandyBreaker(cb breaker.Breaker) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (response interface{}, err error) {
			if !cb.Allow() {
				return nil, breaker.ErrCircuitOpen
			}

			defer func(begin time.Time) {
				if err == nil {
					cb.Success(time.Since(begin))
				} else {
					cb.Failure(time.Since(begin))
				}
			}(time.Now())

			response, err = next(ctx, request)
			return
		}
	}
}
```

## 使用
参考 [examples/set.go](https://github.com/go-kit/kit/blob/master/examples/addsvc/pkg/addendpoint/set.go)
```go
var sumEndpoint endpoint.Endpoint

sumEndpoint = MakeSumEndpoint(svc)
sumEndpoint = ratelimit.NewErroringLimiter(rate.NewLimiter(rate.Every(time.Second), 1))(sumEndpoint)
sumEndpoint = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(sumEndpoint)
```

## 小结
把各个中间件封装成 endpoint.Middleware(`func(next endpoint.Endpoint) endpoint.Endpoint`), 这样就可以通过组合方式把中间件组合起来（这也就是抽象类型 Endpoint 的优势）。


