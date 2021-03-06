### GoCore

A library with a set of standardized functions for applications written in Go at Intercom. Requires Go 1.7+.

To use:

Checkout into your gopath.

Vendor it into your project, making sure dependencies are satisfied (GoCore does not vendor it's own dependencies)

#### Logs

Structured logs in standard LogFmt or JSON format, with levels.

Can use in global logger form:

```go
// don't have to set a namespace, can just import and reference via "log" if you don't need the default logger too.
import corelog "github.com/intercom/gocore/log"

func main() {
  // setup logger before using
  corelog.SetupLogFmtLoggerTo(os.Stderr)

  // or JSON format:
  corelog.SetupJSONLoggerTo(os.Stderr)

  // log messages
  corelog.LogInfoMessage("reading items")

  // structured logging
  corelog.LogInfo("read_item_count", 4, "status", "finished")

  // both
  corelog.LogInfoMessage("reading items", "read_item_count", 4, "status", "finished")

  // setting standard fields to use
  corelog.With("instance_id", "67daf")
}
```

Or with an instance of the logger for passing around:

```go
logger := corelog.JSONLoggerTo(os.Stderr)
logger.LogInfoMessage("foo")
```

#### Metrics

Standardised Metrics options, for Global setup or individual.

```go

import "github.com/intercom/gocore/metrics"

func main() {
  // set the global metrics recorder to a new statsd recorder with namespace
  // without this, global metrics are no-opped.
  statsdRecorder, err := metrics.NewStatsdRecorder("127.0.0.1:8888", "namespace")
  if err == nil {
    metrics.SetMetricsGlobal(statsdRecorder)
  }

  // use global metrics
  metrics.IncrementCount("metricName")
  metrics.MeasureSince("metricName", startTime)

  // set prefix for all global metrics
  metrics.SetPrefix("prefixName")

  // create a new metric instance for separate collections:
  perAppMetrics, _ := metrics.NewStatsdRecorder("127.0.0.1:8889", "per-app-namespace")

  // use same recording methods
  perAppMetrics.IncrementCount("metricName")
}
```

##### Datadog statsd recorder

```go
recorder, _ = metrics.NewDatadogStatsdRecorder("127.0.0.1:8125", "namespace", "hostname")

// individually tagged calls
recorder.WithTag("tagkey", "tagvalue").IncrementCount("metricName")

// re-use a tagged recorder
tagged := recorder.WithTag("tagkey", "tagvalue")
tagged.MeasureSince("metricName", time.Now())
```

To add a new recorder, implement the MetricsRecorder interface.

#### Monitoring

Standardised Monitoring options, for Global setup or individual. Currently, monitoring to Sentry is implemented.

```go
import "github.com/intercom/gocore/monitoring"

func main() {
  // set the global monitoring recorder to a new sentry monitor
  // without this, global monitoring is no-opped.
  sentryMonitor, err := monitoring.NewSentryMonitor("sentryDSN")
  if err == nil {
    monitoring.SetMonitoringGlobal(sentryMonitor)
  }

  // use global monitoring
  monitoring.CaptureException(errors.New("NewError"))

  // create a new monitoring instance:
  sentryMonitoring, _ := monitoring.NewSentryMonitor("sentryDSN")

  // use same capture method
  sentryMonitoring.CaptureException(errors.New("NewError"))
}
```

#### API

The coreapi package ties together the other packages into a set of useful behaviours for constructing API's wired with per-request loggers, metrics and monitoring. Compatible with standard library.

There's a simple wrapper around the default router, to enable easier use of standard middleware. Other routers compatible with standard library should work fine too.

```go
import "github.com/intercom/gocore/coreapi"

func main()  {
	// setup the MiddlewareMux with default logger, recorder, and monitor
	logger := log.LogfmtLoggerTo(os.Stderr)
	recorder, _ := metrics.NewDatadogStatsdRecorder("127.0.0.1:8888", "myservice", "hostname")
	sentryMonitor, _ := monitoring.NewSentryMonitor("sentryDSN")
	mux := coreapi.NewMiddlewareMuxWithDefaults(logger, recorder, sentryMonitor)

	// handle a request to "/user",
	mux.Handle("/user", http.HandlerFunc(User))
	
	// or use some middleware
  middleware := func(http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      coreapi.GetLogger(r).LogInfoMessage("hello from middleware!")
    })
  }
  mux.Use(middleware)
  mux.Handle("/hello", http.HandlerFunc(User))
	
  http.ListenAndServe(":3000", mux)
}

// our http handler
func User(w http.ResponseWriter, r *http.Request) {
  coreapi.GetLogger(r).LogErrorMessage("message") // we have access to a request-scoped logger, which has the URL and request id already set
	coreapi.GetMetrics(r).IncrementCount("request_count")
	coreapi.GetMonitor(r).CaptureException(errors.New("something went wrong"))
}

```

There's also some handy Response objects, that can be used to write formatted data using the http.ResponseWriter.

```go
func User(w http.ResponseWriter, r *http.Request) {
	// json response
	coreapi.JSONResponse(200, &UserResponse{Name: "Foo", Email: "foo@bar.com"}).WriteTo(w)
	
	// or json error
	coreapi.JSONErrorResponse(400, errors.New("bad error :(")).WriteTo(w)

  // or empty:
  coreapi.EmptyResponse(500)
}

type UserResponse struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}
``` 
