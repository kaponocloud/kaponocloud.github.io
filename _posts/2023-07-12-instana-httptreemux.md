---
title: Create Instana Golang intrument for Httptreemux
date: 2023-07-12 16:00:30 -0700
categories: [Go,Instana]
tags: [golang,instana,httptreemux,instrument]
---

## Instrumentation to [httptreemux](https://github.com/dimfeld/httptreemux)

```go
package instahttptreemux

import (
	"net/http"

	mux "github.com/dimfeld/httptreemux/v5"
	instana "github.com/instana/go-sensor"
)

// NewRouter is wrapper for mux.NewRouter()
func NewRouter(sensor instana.TracerLogger) *mux.TreeMux {
	r := mux.New()
	AddMiddleware(sensor, r)

	return r
}

// AddMiddleware instruments the mux.Router instance with Instana
func AddMiddleware(sensor instana.TracerLogger, router *mux.TreeMux) {
	router.UseHandler(func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
			instana.TracingNamedHandlerFunc(sensor, req.RequestURI, req.URL.Path, func(w http.ResponseWriter, req *http.Request) {
				next.ServeHTTP(w, req)
			})(w, req)
		})
	})
}
```

## References

- [Quick start guide for enabling Golang apps to use Instana](https://www.ibm.com/docs/en/instana-observability/current?topic=go-collector-installation)
  - Gather process metrics only

  ```bash
  go get github.com/instana/go-sensor
  ```

  adding the following code at the beginning of your main() function:

  ```go
  instana.InitSensor(instana.DefaultOptions())
  ```
  
  - [Instrument your application](https://www.ibm.com/docs/en/SSE1JP5_current/src/pages/ecosystem/go/howto.html#instrumentation)
    - [Instrument HTTP service](https://www.ibm.com/docs/en/instana-observability/current?topic=go-collector-common-operations#http-services)

    ```go
    sensor := instana.NewSensor("my-http-server")

    http.HandleFunc("/", instana.TracingHandlerFunc(sensor, "/", func(w http.ResponseWriter, req *http.Request) {
        // ...
    }))
    ```

    ```go
    req, err := http.NewRequest("GET", url, nil)
    client := &http.Client{
        Transport: instana.RoundTripper(sensor, nil),
    }

    ctx := instana.ContextWithSpan(context.Background(), parentSpan)
    resp, err := client.Do(req.WithContext(ctx))
    ```

- [Instrumentation modules for 3rd-party frameworks](https://www.ibm.com/docs/en/instana-observability/current?topic=technologies-monitoring-go#supported-frameworks-and-libraries)
- [Instana Git](https://pkg.go.dev/github.com/instana/go-sensor#pkg-overview.in)
