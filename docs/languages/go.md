## Go (Golang)

[Go, aka Golang](https://go.dev/) is a popular open-source programming language created and maintained by a team at Google.

!!! info "Do you need to customise this template?"

    You can customise the official templates, or provide your own. The code for this templates is available on GitHub: [openfaas/golang-http-template](https://github.com/openfaas/golang-http-template/tree/master/template/golang-middleware).

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

We have a separate golang template `golang-otel` that has built-in support OpenTelemetry traces. The template is based on the `golang-middleware` template and allows you to get traces for Golang functions with minimal code changes.

Some of the key benefits of using the `golang-otel` template are:

- **No boilerplate** - Avoid boilerplate code to configure providers and traces in Go functions. No need to fork and maintain your own version of the golang templates.
- **Configuration using environment variables** - Simplify configuration with environment-based settings, reducing the need for code changes.
- **HTTP instrumentation** - Incoming HTTP requests to the function handler are automatically instrumented.
- **Extensibility with custom traces** - Easily add custom spans using the [OpenTelemetry Go Trace API](https://pkg.go.dev/go.opentelemetry.io/otel) without much boilerplate code.

Create a new function with the `golang-otel` template:

```sh
faas-cli new echo --lang golang-otel
```

Use environment variables to configure the traces exporter for the function.

```diff
functions:
  echo:
    lang: golang-otel
    handler: ./echo
    image: echo:latest
+    environment:
+      OTEL_TRACES_EXPORTER: console,otlp
+      OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_EXPORTER_OTLP_ENDPOINT:-collector:4317}
```

The `golang-otel` template supports a subset of [OTEL SDK environment variables](https://opentelemetry.io/docs/languages/sdk-configuration/) to configure the exporter.

- `OTEL_TRACES_EXPORTER` specifies which tracer exporter to use. In this example traces are exported to `console` (stdout) and with `otlp` to send traces to an endpoint that accepts OTLP via gRPC. 
- `OTEL_EXPORTER_OTLP_ENDPOINT` sets the endpoint where telemetry is exported to.
- `OTEL_SERVICE_NAME` sets the name of the service associated with the telemetry and is used to identify telemetry for a specific function. By default `<fn-name>.<fn-namespace>` is used as the service name on Kubernetes or `<fn-name>` when running the function with OpenFaaS Edge, or locally with `faas-cli local-run`.
- `OTEL_EXPORTER_OTLP_TRACES_INSECURE` can be set to true to disable TLS if that is not supported by the OpenTelemetry collector.

The template can be used as a drop-in replacement for functions that already use the `golang-middleware` template to get HTTP invocation traces for your existing functions without any code changes. All that is required is to change the `lang` field in the `stack.yaml` configuration.

```diff
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  echo:
-   lang: golang-middleware
+   lang: golang-otel
    handler: ./echo
    image: echo:latest
```

### Creating custom traces in the function handler

here may be cases where an instrumentation library is not available or you may want to add custom tracing data for some of your own functions and methods.

When using the `golang-otel` template the registered global trace provider can be retrieved to add custom spans in the function handler. A span represents a unit of work or operation. Spans are the building blocks of traces.

> Check out the [OpenTelemetry docs](https://opentelemetry.io/docs/languages/go/instrumentation/#creating-spans) for more information on how to work with spans.

On your code you can call the [otel.Tracer](https://pkg.go.dev/go.opentelemetry.io/otel#Tracer) function to get a named tracer and start a new span.

Make sure to add the required packages to the function handler:

```sh
go get "go.opentelemetry.io/otel"
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

func callOpenAI(ctx context.Context, input []byte) {
	// Get a tracer and create a new span
	ctx,  span := otel.Tracer("function").Start(ctx, "call-openAI")
	defer span.End()

	// Sleep for 2 seconds to simulate some work.
	time.Sleep(time.Second * 2)
}


func Handle(w http.ResponseWriter, r *http.Request) {
	var input []byte

	if r.Body != nil {
		defer r.Body.Close()

		body, _ := io.ReadAll(r.Body)

		input = body
	}

	// Call function with the request context to pass on any parent spans.
	callOpenAI(r.Context(), input)

	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Done processing.")
}
```

To create a new span with a tracer, you’ll need a handle on a `context.Context` instance. These will typically come from the request object and may already contain a parent span from an [instrumentation library](https://opentelemetry.io/docs/languages/go/libraries/).

You can now add [attributes](https://opentelemetry.io/docs/languages/go/instrumentation/#span-attributes) and [events](https://opentelemetry.io/docs/languages/go/instrumentation/#events), [set the status](https://opentelemetry.io/docs/languages/go/instrumentation/#set-span-status) and [record errors](https://opentelemetry.io/docs/languages/go/instrumentation/#record-errors) on the span as required.

## OpenTelemetry zero-code instrumentation

The beta Release for [zero-code instrumentation](https://opentelemetry.io/docs/concepts/instrumentation/zero-code/) of Go applications [has recently been announced](https://opentelemetry.io/blog/2025/go-auto-instrumentation-beta/).

Go auto-instrumentation has not been tested with OpenFaaS functions yet. The easiest way to try it out on Kubernetes is probably by using the [Opentelemetry Operator for Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/).