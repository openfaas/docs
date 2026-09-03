The [Fan-out pattern](/languages/patterns/#fan-out-pattern) can be used to split
a batch into independent items and submit each item to a worker function for
asynchronous processing.

This page builds a deliberately small example: `fan-out` accepts a JSON array
of strings, queues one invocation of `batch-worker` for each item, and returns
the number of submitted items.

```text
        [ Client ]
            │ batch with three items
            ▼
      [ fan-out ] ──► [ Async queue ] ──► [ Queue-worker ]
            │                                  ├── "one" ───► [ batch-worker ]
            │                                  ├── "two" ───► [ batch-worker ]
            │                                  └── "three" ─► [ batch-worker ]
            ▼
[ Response: 202 {"submitted": 3} ]
```

This simple batch highlights several properties of the Fan-out pattern:

* **Batch splitting:** `fan-out` creates one asynchronous invocation for each
  item in the input batch.
* **Asynchronous response:** the caller receives a response after the items
  have been accepted by the queue, without waiting for processing to finish.
* **Independent processing:** items can be processed concurrently and may
  complete in a different order from the input.
* **Independent scaling:** the queue provides back pressure while OpenFaaS can
  scale the `batch-worker` function to handle the load.

## Create the functions

Choose a language and scaffold both functions in one `stack.yaml` file:

=== "Go"

    ```bash
    faas-cli template store pull golang-middleware
    faas-cli new --lang golang-middleware fan-out \
      --prefix ttl.sh/openfaas-examples
    faas-cli new --lang golang-middleware batch-worker \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

=== "Python"

    ```bash
    faas-cli template store pull python3-http
    faas-cli new --lang python3-http fan-out \
      --prefix ttl.sh/openfaas-examples
    faas-cli new --lang python3-http batch-worker \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

=== "Node.js"

    ```bash
    faas-cli template store pull node24
    faas-cli new --lang node24 fan-out \
      --prefix ttl.sh/openfaas-examples
    faas-cli new --lang node24 batch-worker \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

The example uses the public [ttl.sh](https://ttl.sh) registry. Replace the
prefix with your own registry for production use.

The full source code and `stack.yaml` files are available on GitHub for
[Go](https://github.com/openfaas/function-patterns/tree/master/go/fan-out),
[Python](https://github.com/openfaas/function-patterns/tree/master/python/fan-out),
and [Node.js](https://github.com/openfaas/function-patterns/tree/master/node/fan-out).

## Implement the worker

The worker receives one string from the batch and returns a JSON result. A real
worker could transform a record, generate a report, or process a file stored in
object storage.

=== "Go"

    `batch-worker/handler.go`:

    ```go
    package function

    import (
        "encoding/json"
        "io"
        "net/http"
    )

    func Handle(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        item, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, "unable to read item", http.StatusBadRequest)
            return
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]interface{}{
            "item":      string(item),
            "processed": true,
        })
    }
    ```

=== "Python"

    `batch-worker/handler.py`:

    ```python
    def handle(event, context):
        item = (
            event.body.decode()
            if isinstance(event.body, bytes)
            else str(event.body)
        )

        return {
            "statusCode": 200,
            "body": {"item": item, "processed": True},
        }
    ```

=== "Node.js"

    `batch-worker/handler.js`:

    ```javascript
    'use strict'

    module.exports = async (event, context) => {
      const item = Buffer.isBuffer(event.body)
        ? event.body.toString()
        : String(event.body || '')

      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed({ item, processed: true })
    }
    ```

## Implement fan-out

The `fan-out` function decodes the batch and posts each item to the gateway's
`/async-function/batch-worker` route. Each `202 Accepted` response confirms
that an item was queued, not that the worker has completed it.

=== "Go"

    `fan-out/handler.go`:

    ```go
    package function

    import (
        "bytes"
        "context"
        "encoding/json"
        "fmt"
        "net/http"
        "os"
        "strings"
        "time"
    )

    const (
        workerFunction = "batch-worker"
        submitTimeout  = 5 * time.Second
    )

    func Handle(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        var batch []string
        if err := json.NewDecoder(r.Body).Decode(&batch); err != nil {
            http.Error(w, "expected a JSON array of strings", http.StatusBadRequest)
            return
        }
        if len(batch) == 0 {
            http.Error(w, "batch must contain at least one item", http.StatusBadRequest)
            return
        }

        gateway := os.Getenv("gateway_url")
        if gateway == "" {
            gateway = "http://gateway.openfaas:8080"
        }
        endpoint := strings.TrimRight(gateway, "/") +
            "/async-function/" + workerFunction
        callback := r.Header.Get("X-Callback-Url")

        // Submit one asynchronous invocation for each item in the batch.
        for i, item := range batch {
            if err := submit(r.Context(), endpoint, item, callback); err != nil {
                http.Error(w, fmt.Sprintf("item %d: %s", i+1, err), http.StatusBadGateway)
                return
            }
        }

        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusAccepted)
        json.NewEncoder(w).Encode(map[string]int{"submitted": len(batch)})
    }

    func submit(parent context.Context, endpoint, item, callback string) error {
        ctx, cancel := context.WithTimeout(parent, submitTimeout)
        defer cancel()

        req, err := http.NewRequestWithContext(
            ctx,
            http.MethodPost,
            endpoint,
            bytes.NewBufferString(item),
        )
        if err != nil {
            return err
        }
        req.Header.Set("Content-Type", "text/plain")
        if callback != "" {
            req.Header.Set("X-Callback-Url", callback)
        }

        res, err := http.DefaultClient.Do(req)
        if err != nil {
            return err
        }
        defer res.Body.Close()

        if res.StatusCode != http.StatusAccepted {
            return fmt.Errorf("queue returned %s", res.Status)
        }

        return nil
    }
    ```

=== "Python"

    `fan-out/handler.py`:

    ```python
    import json
    import os

    import requests


    WORKER_FUNCTION = "batch-worker"
    SUBMIT_TIMEOUT = 5


    def handle(event, context):
        try:
            batch = json.loads(event.body)
        except (TypeError, ValueError):
            return error(400, "expected a JSON array of strings")

        if (
            not isinstance(batch, list)
            or not batch
            or not all(isinstance(item, str) for item in batch)
        ):
            return error(400, "expected a non-empty JSON array of strings")

        gateway = os.getenv("gateway_url", "http://gateway.openfaas:8080")
        endpoint = f"{gateway.rstrip('/')}/async-function/{WORKER_FUNCTION}"
        callback = event.headers.get("X-Callback-Url", "")

        # Submit one asynchronous invocation for each item in the batch.
        for index, item in enumerate(batch):
            headers = {"Content-Type": "text/plain"}
            if callback:
                headers["X-Callback-Url"] = callback

            try:
                response = requests.post(
                    endpoint,
                    data=item.encode(),
                    headers=headers,
                    timeout=SUBMIT_TIMEOUT,
                )
            except requests.RequestException as err:
                return error(502, f"item {index + 1}: {err}")

            if response.status_code != 202:
                return error(
                    502,
                    f"item {index + 1}: queue returned {response.status_code}",
                )

        return {"statusCode": 202, "body": {"submitted": len(batch)}}


    def error(status_code, message):
        return {"statusCode": status_code, "body": message}
    ```

    Add `requests` to `fan-out/requirements.txt`.

=== "Node.js"

    `fan-out/handler.js`:

    ```javascript
    'use strict'

    const workerFunction = 'batch-worker'
    const submitTimeout = 5000

    module.exports = async (event, context) => {
      let batch
      try {
        batch = JSON.parse(requestBody(event.body))
      } catch (error) {
        return fail(context, 400, 'expected a JSON array of strings')
      }

      if (!Array.isArray(batch) || batch.length === 0 ||
          !batch.every(item => typeof item === 'string')) {
        return fail(context, 400, 'expected a non-empty JSON array of strings')
      }

      const gateway = process.env.gateway_url ||
        'http://gateway.openfaas:8080'
      const endpoint = `${gateway.replace(/\/$/, '')}/async-function/${workerFunction}`
      const callback = String(
        (event.headers || {})['x-callback-url'] || ''
      )

      // Submit one asynchronous invocation for each item in the batch.
      for (const [index, item] of batch.entries()) {
        const headers = { 'Content-Type': 'text/plain' }
        if (callback) {
          headers['X-Callback-Url'] = callback
        }

        let response
        try {
          response = await fetch(endpoint, {
            method: 'POST',
            headers,
            body: item,
            signal: AbortSignal.timeout(submitTimeout)
          })
        } catch (error) {
          return fail(context, 502, `item ${index + 1}: ${error.message}`)
        }

        if (response.status !== 202) {
          return fail(
            context,
            502,
            `item ${index + 1}: queue returned ${response.status}`
          )
        }
      }

      return context
        .status(202)
        .headers({ 'Content-Type': 'application/json' })
        .succeed({ submitted: batch.length })
    }

    function requestBody (body) {
      if (Buffer.isBuffer(body)) {
        return body.toString()
      }
      return String(body || '')
    }

    function fail (context, status, message) {
      return context
        .status(status)
        .headers({ 'Content-Type': 'text/plain' })
        .succeed(message)
    }
    ```

The five-second submission timeout belongs to the `fan-out` implementation in
this example. It limits how long the function waits for the gateway to accept
each item; it does not limit how long `batch-worker` may run after the item has
been queued.

## Configure the functions

=== "Go"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      fan-out:
        lang: golang-middleware
        handler: ./fan-out
        image: ttl.sh/openfaas-examples/fan-out:latest

      batch-worker:
        lang: golang-middleware
        handler: ./batch-worker
        image: ttl.sh/openfaas-examples/batch-worker:latest
    ```

=== "Python"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      fan-out:
        lang: python3-http
        handler: ./fan-out
        image: ttl.sh/openfaas-examples/python-fan-out:latest
        build_args:
          TEST_ENABLED: "true"

      batch-worker:
        lang: python3-http
        handler: ./batch-worker
        image: ttl.sh/openfaas-examples/python-batch-worker:latest
        build_args:
          TEST_ENABLED: "true"
    ```

=== "Node.js"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      fan-out:
        lang: node24
        handler: ./fan-out
        image: ttl.sh/openfaas-examples/node-fan-out:latest

      batch-worker:
        lang: node24
        handler: ./batch-worker
        image: ttl.sh/openfaas-examples/node-batch-worker:latest
    ```

## Deploy and submit a batch

Build and deploy both functions:

```bash
faas-cli up --tag=sha
```

Submit a batch:

```bash
curl -i http://127.0.0.1:8080/function/fan-out \
  -H "Content-Type: application/json" \
  -d '["one", "two", "three"]'
```

The function confirms that all three items have been accepted by the queue:

```text
HTTP/1.1 202 Accepted
Content-Type: application/json

{"submitted":3}
```

The queue-worker then invokes `batch-worker` once for each item. Its response
for the first item is:

```json
{
  "item": "one",
  "processed": true
}
```

By default, responses from asynchronous invocations are discarded. To receive
each result, set `X-Callback-Url` on the request to `fan-out`. The function
forwards the header to each queued invocation, and the queue-worker sends each
result to that endpoint. Callbacks are independent and may arrive out of order.

### Receive callback results

Deploy the `printer` function from the store and follow its logs:

```bash
faas-cli store deploy printer
faas-cli logs printer -t
```

In another terminal, submit the batch with a callback URL:

```bash
curl -s http://127.0.0.1:8080/function/fan-out \
  -H "Content-Type: application/json" \
  -H "X-Callback-Url: http://gateway.openfaas:8080/function/printer" \
  -d '["one", "two", "three"]'
```

The `printer` logs receive one callback per item, with a body similar to:

```json
{
  "item": "one",
  "processed": true
}
```

When one action should run only after the complete batch has finished, use the
[fan-in pattern](/languages/patterns/fan-in/) to collect and combine the individual
results.

## Fan-out considerations

* Batch items must be independent. If one item needs another item's result,
  coordinate them in sequence instead.
* The gateway returns an `X-Call-Id` for each accepted asynchronous
  invocation. Retain these IDs when individual items need to be tracked or
  cancelled with `DELETE /async-function/<call-id>`. See
  [cancel asynchronous invocations](/reference/async/#cancel-async-invocations)
  for more information.
* The queue-worker controls how many items it
  [processes concurrently](/reference/async/#parallelism), and OpenFaaS can
  [autoscale](/architecture/autoscaling/) `batch-worker` when the load increases.
* The queue-worker handles
  [retries for failed invocations](/openfaas-pro/retries/), allowing individual
  batch items to recover from temporary errors.
