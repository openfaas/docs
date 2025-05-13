## Go (Golang)

[Go, aka Golang](https://go.dev/) is a popular open-source programming language created and maintained by a team at Google.

### Official Go template

The official and recommended template for Go for OpenFaaS is `golang-middleware`.

It implements handler for HTTP requests using `http.HandleFunc` from Go's stdlib. It has support for Go modules, vendoring, unit testing, and adding static files.

You can pull this template by running:

```bash
faas-cli template store pull golang-middleware
```

The source for the template is: [https://github.com/openfaas/golang-http-template](https://github.com/openfaas/golang-http-template)

The most comprehensive guide and set of examples for this template are in the Premium and Team Edition of Alex Ellis' book: [Everyday Golang](https://store.openfaas.com/l/everyday-golang).

> This is an official template maintained by OpenFaaS Ltd.

## Create a new function

Create a new function using the template:

```bash
faas-cli new --lang golang-middleware echo
```

The handler will be created as `./echo/handler.go`, along with a `go.mod` for managing dependencies.

`handler.go`:

```go
package function

import (
	"fmt"
	"io"
	"net/http"
)

func Handle(w http.ResponseWriter, r *http.Request) {
	var input []byte

	if r.Body != nil {
		defer r.Body.Close()

		body, _ := io.ReadAll(r.Body)

		input = body
	}

	w.WriteHeader(http.StatusOK)
	w.Write([]byte(fmt.Sprintf("Body: %s", string(input))))
}
```

Just like any other HTTP server written with the Go standard library, the incoming HTTP request is available via the `http.Request` object. The response is written to the `http.ResponseWriter` object. Any body passed in from the client should be closed to avoid any resource leaks.

## Add a dependency

To add a dependency such as a bcrypt library, you can use the `go get` command:

```bash
faas-cli new --lang golang-middleware bcrypt

cd bcrypt

go get golang.org/x/crypto/bcrypt
```

Then write a `bcrypt/handler.go`:

```go
package function

import (
	"fmt"
	"io"
	"net/http"

	"golang.org/x/crypto/bcrypt"
)

func Handle(w http.ResponseWriter, r *http.Request) {
	var input []byte

	if r.Body != nil {
		defer r.Body.Close()
		body, _ := io.ReadAll(r.Body)
		input = body
	}

	res, err := bcrypt.GenerateFromPassword(input, bcrypt.DefaultCost)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "%s", res)
}
```

You'll see that `bcrypt/go.mod` has been updated with the new dependency:

```
module handler/function

go 1.18

require golang.org/x/crypto v0.13.0 // indirect
```

## Working with JSON

Example writing a JSON response from your function.

```go
package function

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
)

func Handle(w http.ResponseWriter, r *http.Request) {
	var input []byte

	if r.Body != nil {
		defer r.Body.Close()

		// read request payload
		reqBody, err := io.ReadAll(r.Body)

		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return

		input = reqBody
		}
	}

	// log to stdout
	fmt.Printf("request body: %s", string(input))

	response := struct {
		Payload     string              `json:"payload"`
		Headers     map[string][]string `json:"headers"`
		Environment []string            `json:"environment"`
	}{
		Payload:     string(input),
		Headers:     r.Header,
		Environment: os.Environ(),
	}

	resBody, err := json.Marshal(response)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

    // write result
	w.WriteHeader(http.StatusOK)
	w.Write(resBody)
}
```

## Use private code

If you have private code to include in your function from another repository, then the easiest way to do this is to use vendoring.

```
faas-cli new --lang golang-middleware secret-fn
```

Create a new function just like in the previous example, when you run `go get`, prefix it with `GOPRIVATE`, and make sure that any `go` commands that you run are performed inside the function's source code directory.

```bash
cd secret-fn
GOPRIVATE=github.com/acmecorp go get
GOPRIVATE=github.com/acmecorp go mod vendor
```

Then you can commit the `vendor` directory to your repository.

You shouldn't need to set GO111MODULE=on, as the template already has this set, but you can do so explicitly if you wish.

```yaml
functions:
  secret-fn:
    lang: golang-middleware
    handler: ./secret-fn
    image: ttl.sh/test/secret-fn:latest
    build_args:
      GO111MODULE: on
```

## Add static files to your function

A common use-case for static files is when you want to serve HTML, lookup information from a JSON manifest or render some kind of templates.

If a folder named `static` is found in the root of your function's source code, **it will be copied** into the final image published for your function.

To read this back at runtime, you can do the following:

```go
package function

import (
    "net/http"
    "os"
)

func Handle(w http.ResponseWriter, r *http.Request) {

    data, err := os.ReadFile("./static/file.txt")

    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }

    w.Write(data)
}
```

## Add your own sub-modules

If you would like to include sub-modules, a certain replace statement is required in your `go.mod` file: `replace handler/function => ./`. This replace statement allows Go to see and use all sub-modules you create with-in your handler, for example

Create your sub-package i.e. `handlers` and run `cd handlers ; go mod init`

Here's handlers/handlers.go:

```golang
package handlers

import (
	"fmt"

	execute "github.com/alexellis/go-execute/pkg/v1"
)

func Handle() {
	ls := execute.ExecTask{
		Command: "exit 1",
		Shell:   true,
	}
	res, err := ls.Execute()
	if err != nil {
		panic(err)
	}

	fmt.Printf("stdout: %q, stderr: %q, exit-code: %d\n", res.Stdout, res.Stderr, res.ExitCode)
}
```

Within your handler.go:

```golang
package function

import (
	"fmt"

	"handler/function/handlers"
)

// Handle a serverless request
func Handle(req []byte) string {

	handlers.Handle()

	return fmt.Sprintf("Hello, Go. You said: %s", string(req))
}
```

Now add the following replace statement to your `go.mod`

```
replace handler/function => ./
```

This can also be affected using

```sh
go mod edit -replace handler/function=./
```

Now you can build with `--build-arg GO111MODULE=on` or with a `build_arg` map entry for the function in its stack.yml.

## Access a database


Example persistent database connection pool between function calls:

```go
package function

import (
	"database/sql"
	"fmt"
	"io"
	"net/http"
	"strings"
	_ "github.com/go-sql-driver/mysql"
)

// db pool shared between function calls
var db *sql.DB

func init() {
	var err error
	db, err = sql.Open("mysql", "user:password@/dbname")
	if err != nil {
		panic(err.Error())
	}

	err = db.Ping()
	if err != nil {
		panic(err.Error())
	}
}

func Handle(w http.ResponseWriter, r *http.Request) {
	var query string
	ctx := r.Context()

	if r.Body != nil {
		defer r.Body.Close()

		// read request payload
		body, err := io.ReadAll(r.Body)

		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		query = string(body)
	}

	// log to stdout
	fmt.Printf("Executing query: %s", query)

	rows, err := db.QueryContext(ctx, query)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer rows.Close()

	ids := make([]string, 0)
	for rows.Next() {
		if e := ctx.Err(); e != nil {
			http.Error(w, e, http.StatusBadRequest)
			return
		}
		var id int
		if err := rows.Scan(&id); err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		ids = append(ids, string(id))
	}
	if err := rows.Err(); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	result := fmt.Sprintf("ids %s", strings.Join(ids, ", "))

	// write result
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(result))
}
```

## Query string example


Example retrieving request query strings

```go
package function
import (
	"fmt"
	"net/http"
)
func Handle(w http.ResponseWriter, r *http.Request) {
	// Parses RawQuery and returns the corresponding
	// values as a map[string][]string
	// for more info https://golang.org/pkg/net/url/#URL.Query
	query := r.URL.Query()
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(fmt.Sprintf("id: %s", query.Get("id"))))
}
```

## OpenTelemetry instrumentation

To instrument a Go function it is recommended to modify one of the existing Go templates. This will allow you to instrument the HTTP server and handle shutdown for the OpenTelemetry SDK properly.

There are two ways to [customise a template](/cli/templates/#how-to-customise-a-template):

- Fork the template repository and modify the template. Recommended method that allows for distribution and reuse of the template.
- Pull the template and apply patches directly in the `./template/<language_name>` directory. Good for quick iteration and experimentation with template modifications. The modified template cannot be shared and reused. Changes may get overwritten when pulling templates again.

In this example we are going to modify the [`golang-middleware` template](https://github.com/openfaas/golang-http-template).

Ensure the right packages are added in the template `go.mod`:

```sh
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/trace \
  go.opentelemetry.io/otel/sdk \
  go.opentelemetry.io/otel/exporters/stdout/stdouttrace \
  go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc \
  go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

Modify the `main.go` file of the template to include code that sets up the OpenTelemetry SDK and instruments the HTTP server.

```diff
func main() {
+	ctx := context.Background()
+
+   // Create exporters by reading the exporter configuration from env variables.
+	exporters, err := getExporters(ctx)
+	if err != nil {
+		log.Fatalf("failed to initialize exporters: %v", err)
+	}
+
+	// Create a new tracer provider with a batch span processor and the given exporters.
+	tp := newTracerProvider(exporters)
+	otel.SetTracerProvider(tp)
+
+	// Handle shutdown properly so nothing leaks.
+	defer func() { _ = tp.Shutdown(ctx) }()
+
	readTimeout := parseIntOrDurationValue(os.Getenv("read_timeout"), defaultTimeout)
	writeTimeout := parseIntOrDurationValue(os.Getenv("write_timeout"), defaultTimeout)
	healthInterval := parseIntOrDurationValue(os.Getenv("healthcheck_interval"), writeTimeout)

	s := &http.Server{
		Addr:           fmt.Sprintf(":%d", 8082),
		ReadTimeout:    readTimeout,
		WriteTimeout:   writeTimeout,
		MaxHeaderBytes: 1 << 20, // Max header of 1MB
	}

- 	http.HandleFunc("/", function.Handle)
+	http.Handle("/", otelhttp.NewHandler(http.HandlerFunc(function.Handle), ""))

	listenUntilShutdown(s, healthInterval, writeTimeout)
}
```

The `getExporters` function creates and returns a list of exporters. The configuration for the exporter is read from environment variables. This makes it easy to later configure the exporters through the `stack.yaml` file of functions that use the template.

By default we support console and OTLP over gRPC exporters but you can modify this function to create your preferred exporters with specific configuration if that is required.

```go
func getExporters(ctx context.Context) ([]sdktrace.SpanExporter, error) {
	var exporters []sdktrace.SpanExporter

	exporterTypes := os.Getenv("OTEL_TRACES_EXPORTER")
	if exporterTypes == "" {
		exporterTypes = "stdout"
	}

	insecureValue := os.Getenv("OTEL_EXPORTER_OTLP_TRACES_INSECURE")
	insecure, err := strconv.ParseBool(insecureValue)
	if err != nil {
		insecure = false
	}

	for _, exp := range strings.Split(exporterTypes, ",") {
		switch exp {
		case "console":
			exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
			if err != nil {
				return nil, err
			}

			exporters = append(exporters, exporter)
		case "otlp":
			endpoint := os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
			if endpoint == "" {
				endpoint = "0.0.0.0:4317"
			}

			opts := []otlptracegrpc.Option{
				otlptracegrpc.WithEndpoint(endpoint),
				otlptracegrpc.WithDialOption(grpc.WithBlock()),
			}

			if insecure {
				opts = append(opts, otlptracegrpc.WithInsecure())

			}

			exporter, err := otlptracegrpc.New(ctx, opts...)
			if err != nil {
				return nil, err
			}

			exporters = append(exporters, exporter)
		default:
			fmt.Printf("unknown OTEL exporter type: %s", exp)
		}
	}

	return exporters, nil
}
```

The `newTracerProvider` initializes a new tracer provider using the provided exporters.

```go
func newTracerProvider(exporters []sdktrace.SpanExporter) *sdktrace.TracerProvider {
	serviceName := os.Getenv("OTEL_SERVICE_NAME")
	if serviceName == "" {
		serviceName = "golang-middleware-function"
	}

	// Ensure default SDK resources and the required service name are set.
	r, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceName(serviceName),
		),
	)

	if err != nil {
		panic(err)
	}

	opts := []sdktrace.TracerProviderOption{
		sdktrace.WithResource(r),
	}

	for _, exporter := range exporters {
		processor := sdktrace.NewBatchSpanProcessor(exporter)
		opts = append(opts, sdktrace.WithSpanProcessor(processor))
	}

	return sdktrace.NewTracerProvider(opts...)
}
```

Use the modified template to create a new function.

```sh
faas-cli new echo --lang golang-middleware-otel
```

Use environment variables to configure the traces exporter for the function.

```diff
functions:
  greet:
    lang: python3-http-otel
    handler: ./greet
    image: greet:latest
+    environment:
+      OTEL_SERVICE_NAME: greet.${NAMESPACE:-openfaas-fn}
+      OTEL_TRACES_EXPORTER: console,otlp
+      OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_EXPORTER_OTLP_ENDPOINT:-collector:4317}
```

- `OTEL_SERVICE_NAME` sets the name of the service associated with the telemetry and is used to identify telemetry for a specific function. It can be set to any value you want but we recommend using the clear function identifier `<fn-name>.<fn-namespace>`.
- `OTEL_TRACES_EXPORTER` specifies which tracer exporter to use. In this example traces are exported to `console` (stdout) and with `otlp` to send traces to an endpoint that accepts OTLP via gRPC.
- `OTEL_EXPORTER_OTLP_ENDPOINT` sets the endpoint where telemetry is exported to.

### Creating custom traces in the function handler

When using a modified OpenTelemetry template the global trace provider can be retrieved to add custom spans or additional instrumentation libraries in the function handler.

Make sure the install the required packages:

```sh
go get "go.opentelemetry.io/otel" \
	"go.opentelemetry.io/otel/trace"
```

Add custom spans in the function handler:

```go
package function

import (
	"fmt"
	"io"
	"net/http"
	"sync"

	"go.opentelemetry.io/otel"
)

var tracer trace.Tracer
var mu sync.Mutex

func getTracer() trace.Tracer {
	if tracer == nil {
		mu.Lock()
		tracer = otel.GetTracerProvider().Tracer("function")
		mu.Unlock()
	}

	return tracer
}


func Handle(w http.ResponseWriter, r *http.Request) {
	ctx, span := getTracer().Start(r.Context(), "function-span")
	defer span.End()

	var input []byte

	if r.Body != nil {
		defer r.Body.Close()

		body, _ := io.ReadAll(r.Body)

		input = body
	}

	w.WriteHeader(http.StatusOK)
	w.Write([]byte(fmt.Sprintf("Body: %s", string(input))))
}
```

The function `otel.GetTracerProvider()` can be used to get the registered trace provider and create a new named tracer for the function.

To create new span with a tracer, you’ll need a handle on a `context.Context` instance. These will typically come from the request object and may already contain a parent span from an [instrumentation library](https://opentelemetry.io/docs/languages/go/libraries/).

Checkout the [OpenTelemetry docs](https://opentelemetry.io/docs/languages/go/instrumentation/) for more information on how to work with spans.

## OpenTelemetry zero-code instrumentation

The beta Release for [zero-code instrumentation](https://opentelemetry.io/docs/concepts/instrumentation/zero-code/) of Go applications [has recently been announced](https://opentelemetry.io/blog/2025/go-auto-instrumentation-beta/).

Go auto-instrumentation has not been tested with OpenFaaS functions yet. The easiest way to try it out on Kubernetes is probably by using the [Opentelemetry Operator for Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/).