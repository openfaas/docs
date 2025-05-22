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

