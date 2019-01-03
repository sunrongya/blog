# go-kit Log & Metrics & Tracing

微服务监控3大核心 Log & Metrics & Tracing

## Log
Log 模块源码有待分析（分析完再补上）

## Metrics
主要是封装 Metrics 接口，及各个 Metrics(Prometheus,InfluxDB,StatsD,expvar) 中间件的封装。

```go
// Counter describes a metric that accumulates values monotonically.
// An example of a counter is the number of received HTTP requests.
type Counter interface {
	With(labelValues ...string) Counter
	Add(delta float64)
}

// Gauge describes a metric that takes specific values over time.
// An example of a gauge is the current depth of a job queue.
type Gauge interface {
	With(labelValues ...string) Gauge
	Set(value float64)
	Add(delta float64)
}

// Histogram describes a metric that takes repeated observations of the same
// kind of thing, and produces a statistical summary of those observations,
// typically expressed as quantiles or buckets. An example of a histogram is
// HTTP request latencies.
type Histogram interface {
	With(labelValues ...string) Histogram
	Observe(value float64)
}
```

## Tracing
Tracing 主要是对 [Zipkin](github.com/openzipkin/zipkin-go), [OpenTracing](github.com/opentracing/opentracing-go), [OpenCensus](go.opencensus.io/trace) 的封装。

### Zipkin
```go
func TraceEndpoint(tracer *zipkin.Tracer, name string) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			var sc model.SpanContext
			if parentSpan := zipkin.SpanFromContext(ctx); parentSpan != nil {
				sc = parentSpan.Context()
			}
			sp := tracer.StartSpan(name, zipkin.Parent(sc))
			defer sp.Finish()

			ctx = zipkin.NewContext(ctx, sp)
			return next(ctx, request)
		}
	}
}
```

### OpenTracing
```go
// TraceServer returns a Middleware that wraps the `next` Endpoint in an
// OpenTracing Span called `operationName`.
//
// If `ctx` already has a Span, it is re-used and the operation name is
// overwritten. If `ctx` does not yet have a Span, one is created here.
func TraceServer(tracer opentracing.Tracer, operationName string) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			serverSpan := opentracing.SpanFromContext(ctx)
			if serverSpan == nil {
				// All we can do is create a new root span.
				serverSpan = tracer.StartSpan(operationName)
			} else {
				serverSpan.SetOperationName(operationName)
			}
			defer serverSpan.Finish()
			otext.SpanKindRPCServer.Set(serverSpan)
			ctx = opentracing.ContextWithSpan(ctx, serverSpan)
			return next(ctx, request)
		}
	}
}

// TraceClient returns a Middleware that wraps the `next` Endpoint in an
// OpenTracing Span called `operationName`.
func TraceClient(tracer opentracing.Tracer, operationName string) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			var clientSpan opentracing.Span
			if parentSpan := opentracing.SpanFromContext(ctx); parentSpan != nil {
				clientSpan = tracer.StartSpan(
					operationName,
					opentracing.ChildOf(parentSpan.Context()),
				)
			} else {
				clientSpan = tracer.StartSpan(operationName)
			}
			defer clientSpan.Finish()
			otext.SpanKindRPCClient.Set(clientSpan)
			ctx = opentracing.ContextWithSpan(ctx, clientSpan)
			return next(ctx, request)
		}
	}
}
```

### OpenCensus
```go
func TraceEndpoint(name string, options ...EndpointOption) endpoint.Middleware {
	if name == "" {
		name = TraceEndpointDefaultName
	}

	cfg := &EndpointOptions{}

	for _, o := range options {
		o(cfg)
	}

	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (response interface{}, err error) {
			ctx, span := trace.StartSpan(ctx, name)
			if len(cfg.Attributes) > 0 {
				span.AddAttributes(cfg.Attributes...)
			}
			defer span.End()

			defer func() {
				if err != nil {
					if lberr, ok := err.(lb.RetryError); ok {
						// handle errors originating from lb.Retry
						attrs := make([]trace.Attribute, 0, len(lberr.RawErrors))
						for idx, rawErr := range lberr.RawErrors {
							attrs = append(attrs, trace.StringAttribute(
								"gokit.retry.error."+strconv.Itoa(idx+1), rawErr.Error(),
							))
						}
						span.AddAttributes(attrs...)
						span.SetStatus(trace.Status{
							Code:    trace.StatusCodeUnknown,
							Message: lberr.Final.Error(),
						})
						return
					}
					// generic error
					span.SetStatus(trace.Status{
						Code:    trace.StatusCodeUnknown,
						Message: err.Error(),
					})
					return
				}

				// test for business error
				if res, ok := response.(endpoint.Failer); ok && res.Failed() != nil {
					span.AddAttributes(
						trace.StringAttribute("gokit.business.error", res.Failed().Error()),
					)
					if cfg.IgnoreBusinessError {
						span.SetStatus(trace.Status{Code: trace.StatusCodeOK})
						return
					}
					// treating business error as real error in span.
					span.SetStatus(trace.Status{
						Code:    trace.StatusCodeUnknown,
						Message: res.Failed().Error(),
					})
					return
				}

				// no errors identified
				span.SetStatus(trace.Status{Code: trace.StatusCodeOK})
			}()
			response, err = next(ctx, request)
			return
		}
	}
}
```

## 使用
参考 [examples/set.go](https://github.com/go-kit/kit/blob/master/examples/addsvc/pkg/addendpoint/set.go)
```go
var concatEndpoint endpoint.Endpoint

concatEndpoint = MakeConcatEndpoint(svc)
concatEndpoint = opentracing.TraceServer(otTracer, "Concat")(concatEndpoint)
concatEndpoint = zipkin.TraceEndpoint(zipkinTracer, "Concat")(concatEndpoint)
concatEndpoint = LoggingMiddleware(log.With(logger, "method", "Concat"))(concatEndpoint)
concatEndpoint = InstrumentingMiddleware(duration.With("method", "Concat"))(concatEndpoint)
```

## 小结
通过把第三方中间件封装成 endpoint.Middleware, 可以与其它 [go-kit](https://github.com/go-kit/kit) 中间件组合。

