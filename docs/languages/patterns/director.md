The [Director pattern](/languages/patterns/#director-pattern) can be used to
implement workflow functions that coordinate other functions through
sequencing, branching, or parallel execution.

This page builds a deliberately small example: two functions each return a
JSON object, and a director combines them into one response.

```text
       [ Client ]
           │
           ▼
┌──────────────────────┐
│       director       │
└──────────┬───────────┘
           ├── 1. invoke ──► [ function-a ] ──► {"a": 1} ──┐
           ├── 2. invoke ──► [ function-b ] ──► {"b": 2} ──┤
           │                                               │
           ├────────────── merge results ◄─────────────────┘
           │
           ▼
[ Response: {"a": 1, "b": 2} ]
```

This simple workflow highlights several properties of the Director pattern:

* **Single endpoint:** the caller only invokes `director` and receives one
  response.
* **Sequencing:** the director invokes `function-a`, followed by `function-b`.
* **Independent functions:** each function is deployed and scaled separately,
  and could be written in a different language.
* **Composition and error handling:** the director validates and merges both
  results, or returns an error when something fails.

## Create the functions

Choose a language and scaffold all three functions in one `stack.yaml` file:

=== "Go"

    ```bash
    faas-cli template store pull golang-middleware
    faas-cli new --lang golang-middleware director \
      --prefix ttl.sh/openfaas-examples
    faas-cli new --lang golang-middleware function-a \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    faas-cli new --lang golang-middleware function-b \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

=== "Python"

    ```bash
    faas-cli template store pull python3-http
    faas-cli new --lang python3-http director \
      --prefix ttl.sh/openfaas-examples
    faas-cli new --lang python3-http function-a \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    faas-cli new --lang python3-http function-b \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

=== "Node.js"

    ```bash
    faas-cli template store pull node24
    faas-cli new --lang node24 director \
      --prefix ttl.sh/openfaas-examples
    faas-cli new --lang node24 function-a \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    faas-cli new --lang node24 function-b \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

The example uses the public [ttl.sh] registry. Replace the
prefix with your own registry for production use.

The full source code and `stack.yaml` files are available on GitHub for
[Go](https://github.com/openfaas/function-patterns/tree/master/go/director),
[Python](https://github.com/openfaas/function-patterns/tree/master/python/director),
and [Node.js](https://github.com/openfaas/function-patterns/tree/master/node/director).

## Implement the two functions

Create two simple functions that each return a JSON payload. Since neither
function needs input, the completed workflow is invoked with a `GET` request.

=== "Go"

    `function-a/handler.go`:

    ```go
    package function

    import (
        "context"
        "encoding/json"
        "net/http"
    )

    func Handle(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]int{"a": 1})
    }
    ```

    `function-b/handler.go`:

    ```go
    package function

    import (
        "encoding/json"
        "net/http"
    )

    func Handle(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]int{"b": 2})
    }
    ```

=== "Python"

    `function-a/handler.py`:

    ```python
    def handle(event, context):
        return {"statusCode": 200, "body": {"a": 1}}
    ```

    `function-b/handler.py`:

    ```python
    def handle(event, context):
        return {"statusCode": 200, "body": {"b": 2}}
    ```

=== "Node.js"

    `function-a/handler.js`:

    ```javascript
    'use strict'

    module.exports = async (event, context) => context
      .status(200)
      .headers({ 'Content-Type': 'application/json' })
      .succeed({ a: 1 })
    ```

    `function-b/handler.js`:

    ```javascript
    'use strict'

    module.exports = async (event, context) => context
      .status(200)
      .headers({ 'Content-Type': 'application/json' })
      .succeed({ b: 2 })
    ```

## Implement the director

The director invokes `function-a` and `function-b` in sequence. It decodes each
response as a JSON object and copies its fields into the combined response.

=== "Go"

    `director/handler.go`:

    ```go
    package function

    import (
        "context"
        "encoding/json"
        "fmt"
        "net/http"
        "os"
        "strings"
        "time"
    )

    const defaultStageTimeout = 5 * time.Second

    func Handle(w http.ResponseWriter, r *http.Request) {
        gateway := os.Getenv("gateway_url")
        if gateway == "" {
            gateway = "http://gateway.openfaas:8080"
        }
        gateway = strings.TrimRight(gateway, "/")

        timeout := stageTimeout()

        // Invoke function-a and decode its JSON response.
        ctxA, cancelA := context.WithTimeout(r.Context(), timeout)
        defer cancelA()
        reqA, err := http.NewRequestWithContext(
            ctxA,
            http.MethodGet,
            gateway+"/function/function-a",
            nil,
        )
        if err != nil {
            http.Error(w, fmt.Sprintf("function-a: %s", err), http.StatusBadGateway)
            return
        }

        resA, err := http.DefaultClient.Do(reqA)
        if err != nil {
            http.Error(w, fmt.Sprintf("function-a: %s", err), http.StatusBadGateway)
            return
        }
        defer resA.Body.Close()
        if resA.StatusCode != http.StatusOK {
            status := resA.StatusCode
            http.Error(w, fmt.Sprintf("function-a: returned %s", resA.Status), status)
            return
        }

        combined := map[string]any{}
        if err := json.NewDecoder(resA.Body).Decode(&combined); err != nil {
            http.Error(
                w,
                fmt.Sprintf("function-a: returned invalid JSON: %s", err),
                http.StatusBadGateway,
            )
            return
        }

        // Invoke function-b after function-a has completed.
        ctxB, cancelB := context.WithTimeout(r.Context(), timeout)
        defer cancelB()
        reqB, err := http.NewRequestWithContext(
            ctxB,
            http.MethodGet,
            gateway+"/function/function-b",
            nil,
        )
        if err != nil {
            http.Error(w, fmt.Sprintf("function-b: %s", err), http.StatusBadGateway)
            return
        }

        resB, err := http.DefaultClient.Do(reqB)
        if err != nil {
            http.Error(w, fmt.Sprintf("function-b: %s", err), http.StatusBadGateway)
            return
        }
        defer resB.Body.Close()
        if resB.StatusCode != http.StatusOK {
            status := resB.StatusCode
            http.Error(w, fmt.Sprintf("function-b: returned %s", resB.Status), status)
            return
        }

        functionB := map[string]any{}
        if err := json.NewDecoder(resB.Body).Decode(&functionB); err != nil {
            http.Error(
                w,
                fmt.Sprintf("function-b: returned invalid JSON: %s", err),
                http.StatusBadGateway,
            )
            return
        }

        // Merge function-b into function-a and return the combined object.
        for key, value := range functionB {
            combined[key] = value
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(combined)
    }

    func stageTimeout() time.Duration {
        if value := os.Getenv("stage_timeout"); value != "" {
            if timeout, err := time.ParseDuration(value); err == nil && timeout > 0 {
                return timeout
            }
        }
        return defaultStageTimeout
    }
    ```

=== "Python"

    `director/handler.py`:

    ```python
    import os

    import requests


    GATEWAY_URL = os.getenv(
        "gateway_url", "http://gateway.openfaas:8080"
    ).rstrip("/")
    STAGE_TIMEOUT = float(os.getenv("stage_timeout", "5"))


    def handle(event, context):
        # Invoke function-a and decode its JSON response.
        try:
            response_a = requests.get(
                f"{GATEWAY_URL}/function/function-a",
                timeout=STAGE_TIMEOUT,
            )
        except requests.RequestException as err:
            return error(502, f"function-a: {err}")

        if response_a.status_code != 200:
            return error(response_a.status_code, f"function-a: {response_a.text}")

        try:
            combined = response_a.json()
        except ValueError as err:
            return error(502, f"function-a returned invalid JSON: {err}")

        if not isinstance(combined, dict):
            return error(502, "function-a did not return a JSON object")

        # Invoke function-b after function-a has completed.
        try:
            response_b = requests.get(
                f"{GATEWAY_URL}/function/function-b",
                timeout=STAGE_TIMEOUT,
            )
        except requests.RequestException as err:
            return error(502, f"function-b: {err}")

        if response_b.status_code != 200:
            return error(response_b.status_code, f"function-b: {response_b.text}")

        try:
            function_b = response_b.json()
        except ValueError as err:
            return error(502, f"function-b returned invalid JSON: {err}")

        if not isinstance(function_b, dict):
            return error(502, "function-b did not return a JSON object")

        # Merge function-b into function-a and return the combined object.
        combined.update(function_b)

        return {"statusCode": 200, "body": combined}


    def error(status_code, message):
        return {"statusCode": status_code, "body": message.strip()}
    ```

    Add `requests` to `director/requirements.txt`.

=== "Node.js"

    `director/handler.js`:

    ```javascript
    'use strict'

    const gatewayURL = process.env.gateway_url ||
      'http://gateway.openfaas:8080'
    const stageTimeout = Number(process.env.stage_timeout || '5') * 1000

    module.exports = async (event, context) => {
      const gateway = gatewayURL.replace(/\/$/, '')

      // Invoke function-a and decode its JSON response.
      let responseA
      try {
        responseA = await fetch(`${gateway}/function/function-a`, {
          method: 'GET',
          signal: AbortSignal.timeout(stageTimeout)
        })
      } catch (error) {
        return fail(context, 502, `function-a: ${error.message}`)
      }

      if (responseA.status !== 200) {
        return fail(context, responseA.status, `function-a: ${await responseA.text()}`)
      }

      let combined
      try {
        combined = await responseA.json()
      } catch (error) {
        return fail(context, 502, `function-a returned invalid JSON: ${error.message}`)
      }

      if (!combined || Array.isArray(combined) || typeof combined !== 'object') {
        return fail(context, 502, 'function-a did not return a JSON object')
      }

      // Invoke function-b after function-a has completed.
      let responseB
      try {
        responseB = await fetch(`${gateway}/function/function-b`, {
          method: 'GET',
          signal: AbortSignal.timeout(stageTimeout)
        })
      } catch (error) {
        return fail(context, 502, `function-b: ${error.message}`)
      }

      if (responseB.status !== 200) {
        return fail(context, responseB.status, `function-b: ${await responseB.text()}`)
      }

      let functionB
      try {
        functionB = await responseB.json()
      } catch (error) {
        return fail(context, 502, `function-b returned invalid JSON: ${error.message}`)
      }

      if (!functionB || Array.isArray(functionB) || typeof functionB !== 'object') {
        return fail(context, 502, 'function-b did not return a JSON object')
      }

      // Merge function-b into function-a and return the combined object.
      Object.assign(combined, functionB)

      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed(combined)
    }

    function fail (context, status, message) {
      return context
        .status(status)
        .headers({ 'Content-Type': 'text/plain' })
        .succeed(message.trim())
    }
    ```

The director calls each function through the gateway, so the functions can be
written in different languages and scaled independently. A transport error or
invalid JSON response returns `502 Bad Gateway`. A non-200 response is
attributed to the function that returned it and its status is passed through to
the caller.

Configure the director's timeouts in `stack.yaml`:

=== "Go"

    ```yaml
    functions:
      director:
        lang: golang-middleware
        handler: ./director
        image: ttl.sh/openfaas-examples/director:latest
        environment:
          stage_timeout: 5s
          exec_timeout: 15s
          read_timeout: 16s
          write_timeout: 16s
    ```

=== "Python"

    ```yaml
    functions:
      director:
        lang: python3-http
        handler: ./director
        image: ttl.sh/openfaas-examples/director:latest
        environment:
          stage_timeout: "5"
          exec_timeout: 15s
          read_timeout: 16s
          write_timeout: 16s
    ```

=== "Node.js"

    ```yaml
    functions:
      director:
        lang: node24
        handler: ./director
        image: ttl.sh/openfaas-examples/director:latest
        environment:
          stage_timeout: "5"
          exec_timeout: 15s
          read_timeout: 16s
          write_timeout: 16s
    ```

## Configure timeouts

A director stays active while it waits for the functions it invokes, so each
call should have a timeout.

This example uses the custom `stage_timeout`
environment variable, the other variables below are OpenFaaS watchdog settings.

| Setting | Scope |
|---------|-------|
| `stage_timeout` | Custom HTTP client timeout for each downstream call |
| `exec_timeout` | Maximum duration of the complete director invocation |
| `read_timeout`, `write_timeout` | Watchdog timeouts, set slightly longer than `exec_timeout` |

This director calls the functions sequentially, so its expected duration is
the sum of both calls plus a small amount of overhead. Configure the director's
timeout for that complete path, not for a single function.

The gateway's `upstream_timeout` must be at least as long as the director's
`exec_timeout`. Adjust the example values for your own functions and see
[extended timeouts](/tutorials/expanded-timeouts/) for the complete
configuration.

## Deploy and invoke

Build, push, and deploy all three functions:

```bash
faas-cli up --tag=sha
```

Invoke the director:

```bash
curl -s http://127.0.0.1:8080/function/director | jq
```

The director returns the union of the two responses:

```json
{
  "a": 1,
  "b": 2
}
```

## Workflow considerations

* Invoke dependent functions in sequence. Independent functions can run in
  parallel to reduce latency, but the director must still wait for every result
  it needs before continuing.
* The director owns the error policy. Depending on the workflow, it can stop,
  retry, return a partial result, or save progress for a later invocation.
* If retrying the director could repeat side effects in a function that already
  completed, make those operations idempotent.
* The functions invoked by a director are deployed independently. Each can use
  a different language, scale separately, and be updated without moving the
  workflow logic out of the director.
* For a long-running workflow, the director can be invoke asynchronously through
  `/async-function/director` with an `X-Callback-Url`. The director continues to
  wait for its functions, while the client receives the final result through
  the callback. See [asynchronous functions](/reference/async/#how-it-works).
