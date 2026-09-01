The [Fan-out pattern](/languages/patterns/#fan-out-pattern) splits a larger
task into smaller, independent items that can be processed in parallel.

Use-cases:

* Processing a batch of independent items without keeping the caller waiting
* Absorbing bursts of work through a queue and processing them as capacity
  becomes available
* Scaling the target function independently and sending each result to a
  callback endpoint

This page implements the pattern as a batch of URL health checks. The
`fan-out` function accepts one trusted URL per line and submits each URL as an
asynchronous invocation of the `url-check` function:

```text
       [ Client ]
            │
            ▼ POST /function/fan-out
┌──────────────────────┐
│       fan-out        │ ◄── splits the batch and returns a summary
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│     queue-worker     │ ◄── drains the queue as capacity becomes available
└──────────┬───────────┘
           ├── URL 1 / async ──► [ url-check ] ──┐
           ├── URL 2 / async ──► [ url-check ] ──┤
           └── URL N / async ──► [ url-check ] ──┘
                                                 │ optional callback
                                                 ▼
                                        [ Result endpoint ]
```

The example demonstrates three parts of fan-out:

* **Submission:** the caller receives call IDs without waiting for the URL
  checks to finish.
* **Queued processing:** the queue-worker invokes `url-check` as capacity
  becomes available, and OpenFaaS can scale the function across replicas.
* **Result delivery:** an optional callback URL receives each health-check
  result independently.

## Create the functions

Choose a language and scaffold both functions in a single `stack.yaml` file:

=== "Go"

    ```bash
    faas-cli template store pull golang-middleware

    faas-cli new --lang golang-middleware fan-out \
      --prefix ttl.sh/openfaas-examples

    faas-cli new --lang golang-middleware url-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

    Replace the generated handler files and `stack.yaml` with the Go files from
    the implementation section below.

=== "Python"

    ```bash
    faas-cli template store pull python3-http

    faas-cli new --lang python3-http fan-out \
      --prefix ttl.sh/openfaas-examples

    faas-cli new --lang python3-http url-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

    Replace the generated handler files, both `requirements.txt` files, and
    `stack.yaml` with the Python files from the implementation section below.

=== "Node.js"

    ```bash
    faas-cli template store pull node24

    faas-cli new --lang node24 fan-out \
      --prefix ttl.sh/openfaas-examples

    faas-cli new --lang node24 url-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

    Replace the generated handler files and `stack.yaml` with the Node.js files
    from the implementation section below.

The example uses the public [ttl.sh](https://ttl.sh) registry. Replace the
prefix with your own registry for production use.

## Implement the functions

### Submitting function: fan-out

=== "Go"

    `fan-out/handler.go`:

    ```go
    package function

    import (
        "bytes"
        "context"
        "encoding/json"
        "fmt"
        "io"
        "net/http"
        "os"
        "strings"
        "time"
    )

    const (
        targetFunction = "url-check"
        submitTimeout  = 30 * time.Second
    )

    type Response struct {
        Submitted int      `json:"submitted"`
        Function  string   `json:"function"`
        Callback  bool     `json:"callback"`
        CallIDs   []string `json:"call_ids,omitempty"`
    }

    // Handle takes a HTTP request body and splits it into one record per line.
    // Each record is submitted as an asynchronous invocation of the target
    // function, then a summary is returned to the caller without waiting for
    // the function invocations to complete.
    func Handle(w http.ResponseWriter, r *http.Request) {
        input, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, "unable to read request body", http.StatusBadRequest)
            return
        }
        defer r.Body.Close()

        gateway := os.Getenv("gateway_url")
        if gateway == "" {
            gateway = "http://gateway.openfaas:8080"
        }

        // Forward the callback URL to every asynchronous invocation. A header on
        // the batch request overrides the environment variable.
        callback := strings.TrimSpace(r.Header.Get("X-Callback-Url"))
        if callback == "" {
            callback = strings.TrimSpace(os.Getenv("callback_url"))
        }

        records := recordsFromInput(string(input))
        if len(records) == 0 {
            http.Error(
                w,
                "expected one record per line in the request body",
                http.StatusBadRequest,
            )
            return
        }

        submitted := 0
        var callIDs []string

        for i, record := range records {
            callID, err := submit(
                r.Context(), gateway, targetFunction, record, callback,
            )
            if err != nil {
                message := fmt.Sprintf(
                    "record %d of %d: %s",
                    i+1,
                    len(records),
                    err,
                )
                http.Error(w, message, http.StatusBadGateway)
                return
            }

            submitted++
            if callID != "" {
                callIDs = append(callIDs, callID)
            }
        }

        res := Response{
            Submitted: submitted,
            Function:  targetFunction,
            Callback:  callback != "",
            CallIDs:   callIDs,
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(res)
    }

    func recordsFromInput(input string) []string {
        var records []string

        for _, record := range strings.Split(strings.TrimSpace(input), "\n") {
            record = strings.TrimSpace(record)
            if record != "" {
                records = append(records, record)
            }
        }

        return records
    }

    func submit(
        ctx context.Context,
        gateway string,
        targetFunction string,
        record string,
        callback string,
    ) (string, error) {
        ctx, cancel := context.WithTimeout(ctx, submitTimeout)
        defer cancel()

        url := strings.TrimRight(gateway, "/") + "/async-function/" + targetFunction
        req, err := http.NewRequestWithContext(
            ctx,
            http.MethodPost,
            url,
            bytes.NewReader([]byte(record)),
        )
        if err != nil {
            return "", fmt.Errorf("unable to invoke %s: %w", targetFunction, err)
        }
        req.Header.Set("Content-Type", "text/plain")
        if callback != "" {
            req.Header.Set("X-Callback-Url", callback)
        }

        res, err := http.DefaultClient.Do(req)
        if err != nil {
            return "", fmt.Errorf("error invoking %s: %w", targetFunction, err)
        }
        defer res.Body.Close()

        if res.StatusCode != http.StatusAccepted {
            out, err := io.ReadAll(res.Body)
            if err != nil {
                return "", fmt.Errorf(
                    "unexpected status %d from %s",
                    res.StatusCode,
                    targetFunction,
                )
            }

            return "", fmt.Errorf(
                "unexpected status %d from %s: %s",
                res.StatusCode,
                targetFunction,
                string(out),
            )
        }

        // the X-Call-Id header can be used to track or cancel the record
        return res.Header.Get("X-Call-Id"), nil
    }
    ```

=== "Python"

    `fan-out/handler.py`:

    ```python
    import os

    import requests


    TARGET_FUNCTION = "url-check"
    SUBMIT_TIMEOUT = 30


    def handle(event, context):
        body = (
            event.body.decode()
            if isinstance(event.body, bytes)
            else str(event.body)
        )
        records = [
            record.strip()
            for record in body.strip().splitlines()
            if record.strip()
        ]
        if not records:
            return error(400, "expected one record per line in the request body")

        gateway = os.getenv("gateway_url", "http://gateway.openfaas:8080")
        callback = event.headers.get("X-Callback-Url", "").strip()
        if not callback:
            callback = os.getenv("callback_url", "").strip()

        call_ids = []
        for index, record in enumerate(records):
            try:
                call_id = submit(gateway, record, callback)
            except requests.RequestException as err:
                return error(502, f"record {index + 1} of {len(records)}: {err}")
            except RuntimeError as err:
                return error(502, f"record {index + 1} of {len(records)}: {err}")

            if call_id:
                call_ids.append(call_id)

        response = {
            "submitted": len(records),
            "function": TARGET_FUNCTION,
            "callback": bool(callback),
        }
        if call_ids:
            response["call_ids"] = call_ids

        return {"statusCode": 200, "body": response}


    def submit(gateway, record, callback):
        headers = {"Content-Type": "text/plain"}
        if callback:
            headers["X-Callback-Url"] = callback

        response = requests.post(
            f"{gateway.rstrip('/')}/async-function/{TARGET_FUNCTION}",
            data=record.encode(),
            headers=headers,
            timeout=SUBMIT_TIMEOUT,
        )
        if response.status_code != 202:
            raise RuntimeError(
                f"unexpected status {response.status_code} "
                f"from {TARGET_FUNCTION}: {response.text}"
            )

        return response.headers.get("X-Call-Id", "")


    def error(status_code, message):
        return {"statusCode": status_code, "body": message}
    ```

    `fan-out/requirements.txt`:

    ```text
    requests
    ```

=== "Node.js"

    `fan-out/handler.js`:

    ```javascript
    'use strict'

    const targetFunction = 'url-check'
    const submitTimeout = 30000

    module.exports = async (event, context) => {
      const input = requestBody(event.body)
      const records = input
        .trim()
        .split('\n')
        .map((record) => record.trim())
        .filter(Boolean)

      if (records.length === 0) {
        return fail(
          context,
          400,
          'expected one record per line in the request body'
        )
      }

      const gateway = process.env.gateway_url ||
        'http://gateway.openfaas:8080'
      const headers = event.headers || {}
      const callback = String(
        headers['x-callback-url'] || process.env.callback_url || ''
      ).trim()

      const callIDs = []
      for (const [index, record] of records.entries()) {
        let callID
        try {
          callID = await submit(gateway, record, callback)
        } catch (error) {
          return fail(
            context,
            502,
            `record ${index + 1} of ${records.length}: ${error.message}`
          )
        }
        if (callID) {
          callIDs.push(callID)
        }
      }

      const response = {
        submitted: records.length,
        function: targetFunction,
        callback: Boolean(callback)
      }
      if (callIDs.length > 0) {
        response.call_ids = callIDs
      }

      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed(response)
    }

    async function submit (gateway, record, callback) {
      const headers = { 'Content-Type': 'text/plain' }
      if (callback) {
        headers['X-Callback-Url'] = callback
      }

      const baseURL = gateway.replace(/\/$/, '')
      const response = await fetch(
        `${baseURL}/async-function/${targetFunction}`,
        {
          method: 'POST',
          headers,
          body: record,
          signal: AbortSignal.timeout(submitTimeout)
        }
      )

      if (response.status !== 202) {
        const body = await response.text()
        throw new Error(
          `unexpected status ${response.status} ` +
          `from ${targetFunction}: ${body}`
        )
      }

      return response.headers.get('X-Call-Id') || ''
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

The submitting function:

* Uses the in-cluster gateway URL by default and submits each URL through
  `/async-function/url-check`.
* Forwards `X-Callback-Url` from the batch request to every asynchronous
  invocation. The `callback_url` environment variable can provide a default.
* Returns the `X-Call-Id` from each accepted submission so individual checks
  can be tracked or cancelled.
* Stops and returns `502 Bad Gateway` if the gateway does not accept one of the
  submissions. Checks accepted before that failure remain queued.

### Fanned-out function: url-check

The function performs an HTTP `GET` with a configurable timeout and returns a
structured health result.

=== "Go"

    `url-check/handler.go`:

    ```go
    package function

    import (
        "context"
        "encoding/json"
        "fmt"
        "io"
        "net/http"
        "net/url"
        "os"
        "strings"
        "time"
    )

    const (
        defaultRequestTimeout = 5 * time.Second
        maxURLLength          = 4096
    )

    var requestTimeout = defaultRequestTimeout

    func init() {
        value := os.Getenv("request_timeout")
        if value == "" {
            return
        }

        timeout, err := time.ParseDuration(value)
        if err != nil || timeout <= 0 {
            panic(fmt.Sprintf("invalid request_timeout %q", value))
        }

        requestTimeout = timeout
    }

    type Response struct {
        URL         string `json:"url"`
        Reachable   bool   `json:"reachable"`
        Healthy     bool   `json:"healthy"`
        StatusCode  int    `json:"status_code,omitempty"`
        ContentType string `json:"content_type,omitempty"`
        DurationMs  int64  `json:"duration_ms"`
        Error       string `json:"error,omitempty"`
    }

    func Handle(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        input, err := io.ReadAll(io.LimitReader(r.Body, maxURLLength+1))
        if err != nil {
            http.Error(w, "unable to read request body", http.StatusBadRequest)
            return
        }
        if len(input) > maxURLLength {
            http.Error(w, "URL is too long", http.StatusBadRequest)
            return
        }

        target := strings.TrimSpace(string(input))
        parsed, err := url.ParseRequestURI(target)
        if err != nil || parsed.Host == "" {
            http.Error(
                w,
                "expected an absolute HTTP or HTTPS URL",
                http.StatusBadRequest,
            )
            return
        }
        if parsed.Scheme != "http" && parsed.Scheme != "https" {
            http.Error(
                w,
                "expected an absolute HTTP or HTTPS URL",
                http.StatusBadRequest,
            )
            return
        }

        start := time.Now()
        ctx, cancel := context.WithTimeout(r.Context(), requestTimeout)
        defer cancel()

        req, err := http.NewRequestWithContext(ctx, http.MethodGet, target, nil)
        if err != nil {
            http.Error(
                w,
                "unable to create health-check request",
                http.StatusBadRequest,
            )
            return
        }
        req.Header.Set("User-Agent", "OpenFaaS URL health check")

        res, requestErr := http.DefaultClient.Do(req)
        result := Response{
            URL:        target,
            DurationMs: time.Since(start).Milliseconds(),
        }
        if requestErr != nil {
            result.Error = requestErr.Error()
            writeJSON(w, result)
            return
        }
        defer res.Body.Close()
        io.Copy(io.Discard, io.LimitReader(res.Body, 1024))

        result.Reachable = true
        result.Healthy = res.StatusCode >= 200 && res.StatusCode < 400
        result.StatusCode = res.StatusCode
        result.ContentType = res.Header.Get("Content-Type")
        writeJSON(w, result)
    }

    func writeJSON(w http.ResponseWriter, result Response) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(result)
    }
    ```

=== "Python"

    `url-check/handler.py`:

    ```python
    import os
    import time
    from urllib.parse import urlparse

    import requests


    MAX_URL_LENGTH = 4096
    REQUEST_TIMEOUT = float(os.getenv("request_timeout", "5"))
    if REQUEST_TIMEOUT <= 0:
        raise ValueError("request_timeout must be greater than zero")


    def handle(event, context):
        body = (
            event.body
            if isinstance(event.body, bytes)
            else str(event.body).encode()
        )
        if len(body) > MAX_URL_LENGTH:
            return error("URL is too long")

        target = body.decode().strip()
        parsed = urlparse(target)
        if parsed.scheme not in ("http", "https") or not parsed.netloc:
            return error("expected an absolute HTTP or HTTPS URL")

        started = time.monotonic()
        result = {
            "url": target,
            "reachable": False,
            "healthy": False,
        }

        try:
            with requests.get(
                target,
                headers={"User-Agent": "OpenFaaS URL health check"},
                timeout=REQUEST_TIMEOUT,
                stream=True,
            ) as response:
                response.raw.read(1024)
                result.update(
                    {
                        "reachable": True,
                        "healthy": 200 <= response.status_code < 400,
                        "status_code": response.status_code,
                        "content_type": response.headers.get("Content-Type", ""),
                        "duration_ms": int((time.monotonic() - started) * 1000),
                    }
                )
        except requests.RequestException as err:
            result["duration_ms"] = int((time.monotonic() - started) * 1000)
            result["error"] = str(err)
            return {"statusCode": 200, "body": result}

        return {"statusCode": 200, "body": result}


    def error(message):
        return {"statusCode": 400, "body": message}
    ```

    `url-check/requirements.txt`:

    ```text
    requests
    ```

=== "Node.js"

    `url-check/handler.js`:

    ```javascript
    'use strict'

    const { performance } = require('node:perf_hooks')

    const maxURLLength = 4096
    const requestTimeout = configuredTimeout(
      process.env.request_timeout || '5',
      'request_timeout'
    )

    module.exports = async (event, context) => {
      const target = requestBody(event.body).trim()
      if (Buffer.byteLength(target) > maxURLLength) {
        return fail(context, 'URL is too long')
      }

      let parsed
      try {
        parsed = new URL(target)
      } catch (error) {
        return fail(context, 'expected an absolute HTTP or HTTPS URL')
      }
      if (parsed.protocol !== 'http:' && parsed.protocol !== 'https:') {
        return fail(context, 'expected an absolute HTTP or HTTPS URL')
      }

      const started = performance.now()
      const result = {
        url: target,
        reachable: false,
        healthy: false
      }

      let response
      try {
        response = await fetch(target, {
          headers: { 'User-Agent': 'OpenFaaS URL health check' },
          signal: AbortSignal.timeout(requestTimeout)
        })
      } catch (error) {
        result.duration_ms = Math.round(performance.now() - started)
        result.error = error.message
        return succeed(context, result)
      }

      if (response.body) {
        await response.body.cancel()
      }
      result.reachable = true
      result.healthy = response.status >= 200 && response.status < 400
      result.status_code = response.status
      result.content_type = response.headers.get('Content-Type') || ''
      result.duration_ms = Math.round(performance.now() - started)
      return succeed(context, result)
    }

    function requestBody (body) {
      if (Buffer.isBuffer(body)) {
        return body.toString()
      }
      return String(body || '')
    }

    function configuredTimeout (value, name) {
      const seconds = Number(value)
      if (!Number.isFinite(seconds) || seconds <= 0) {
        throw new Error(`${name} must be greater than zero`)
      }
      return seconds * 1000
    }

    function succeed (context, body) {
      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed(body)
    }

    function fail (context, message) {
      return context
        .status(400)
        .headers({ 'Content-Type': 'text/plain' })
        .succeed(message)
    }
    ```

Each URL produces a structured result, including unreachable targets and
timeouts, so every outcome can be delivered to the callback endpoint.

!!! warning

    Only submit URLs from a trusted source. Fetching arbitrary user-provided
    URLs can expose internal services through server-side request forgery
    (SSRF). For a public endpoint, enforce an allow-list and validate resolved
    addresses before making the request.

### Stack file

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

      url-check:
        lang: golang-middleware
        handler: ./url-check
        image: ttl.sh/openfaas-examples/url-check:latest
        environment:
          request_timeout: 5s
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

      url-check:
        lang: python3-http
        handler: ./url-check
        image: ttl.sh/openfaas-examples/python-url-check:latest
        build_args:
          TEST_ENABLED: "true"
        environment:
          request_timeout: "5"
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

      url-check:
        lang: node24
        handler: ./url-check
        image: ttl.sh/openfaas-examples/node-url-check:latest
        environment:
          request_timeout: "5"
    ```

## Deploy and submit a batch

Build, push, and deploy both functions:

```bash
faas-cli up --tag=sha
```

The handler splits the request body on newlines, so use `--data-binary` with `curl`:

```bash
printf 'https://www.openfaas.com/\nhttps://docs.openfaas.com/\n' | \
  curl -s --data-binary @- -H "Content-Type: text/plain" \
  http://127.0.0.1:8080/function/fan-out | jq
```

The response confirms that both checks were accepted without waiting for them
to finish:

```json
{
  "submitted": 2,
  "function": "url-check",
  "callback": false,
  "call_ids": [
    "9c0b1a12-fdea-4f01-baff-c5d9f50435ea",
    "4111d512-cdf3-4b8f-96b3-1b7f1f376bd7"
  ]
}
```

## Collect individual results with a callback

By default, the queue-worker discards the response from each `url-check`
invocation. To receive the responses, set `X-Callback-Url` on the request to
`fan-out`. The submitting function copies that URL to every queued invocation,
and the queue-worker posts each result to the callback endpoint.

Callbacks are independent and may arrive in a different order from the input.
This example delivers each result but does not wait for or combine the whole
batch. When that is required, the callback endpoint can use shared storage to
track progress and [fan the results back in](https://www.openfaas.com/blog/fan-out-and-back-in-using-functions/).

### Receive callback results

Deploy the `printer` function from the store and follow its logs:

```bash
faas-cli store deploy printer
faas-cli logs printer -t
```

In another terminal, submit the batch with a callback URL:

```bash
printf 'https://www.openfaas.com/\nhttps://docs.openfaas.com/\n' | \
  curl -s --data-binary @- \
  -H "Content-Type: text/plain" \
  -H "X-Callback-Url: http://gateway.openfaas:8080/function/printer" \
  http://127.0.0.1:8080/function/fan-out | jq
```

The batch response now contains `"callback": true`. The `printer` logs receive
one callback per URL, with a body similar to:

```json
{
  "url": "https://www.openfaas.com/",
  "reachable": true,
  "healthy": true,
  "status_code": 200,
  "content_type": "text/html; charset=utf-8",
  "duration_ms": 84
}
```

The `printer` function is useful for demonstrating callback delivery. Replace
it with an application endpoint when results need to be persisted or acted on.

## Track and cancel checks

Each call ID in the batch response identifies one queued check. Cancel it with
a `DELETE` request to the async endpoint:

```bash
curl -i -X DELETE \
  http://127.0.0.1:8080/async-function/9c0b1a12-fdea-4f01-baff-c5d9f50435ea
```

A `202 Accepted` response indicates that the cancellation request was
accepted. See [asynchronous functions](/reference/async/) for the complete
lifecycle.

## Operational considerations

* The queue-worker processes records up to its configured `max_inflight`
  concurrency, while OpenFaaS can
  [autoscale the function](/architecture/autoscaling/) across replicas. See
  [parallelism](/reference/async/#parallelism).
* A target that cannot be reached produces a successful function invocation
  with `"reachable": false`. This allows the failure result to reach the
  callback rather than being retried as a function error.
* Queue-worker retries apply when the function invocation itself fails. See
  [retries](/openfaas-pro/retries/).
* The maximum payload size for each queued item is 1MB. For larger inputs,
  store the data externally and submit an identifier. See
  [configuration and limits](/reference/async/#configuration-limits).
