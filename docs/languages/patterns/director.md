The [Director pattern](/languages/patterns/#director-pattern) implements a
workflow directly in an OpenFaaS function. The director controls how other
functions are invoked, passes data between them, combines results, and handles
errors.

Use-cases:

* Exposing a single endpoint for an operation that involves several functions
* Hiding the coordination between functions from the caller
* Coordinating functions that are developed, deployed, or scaled independently
* Keeping workflow decisions, error handling, and the final response in one
  place

This page implements the pattern as a telemetry workflow. The
`telemetry-workflow` function validates a sensor reading, runs temperature and
battery checks in parallel, and returns an `ok` or `alert` result:

```text
       [ Client ]
            │
            ▼ POST /function/telemetry-workflow
┌──────────────────────┐
│  telemetry-workflow  │ ◄── director: owns the workflow and handles errors
└──────────┬───────────┘
           │
           ├── 1. Invoke ──► [ validate-reading ]
           │◄──── validated reading ────────────┘
           │
           ├── 2. Invoke in parallel
           │       ├──► [ temperature-check ] ──┐
           │       └──► [ battery-check ] ──────┤
           │◄──── check results ────────────────┘
           │
           └── 3. Combine results and return ok or alert
```

The workflow demonstrates three common Director operations:

* **Sequence:** validation completes before any checks are started.
* **Parallel execution:** the temperature and battery checks run concurrently.
* **Conditional result:** the director returns `alert` when either check raises
  an alert, otherwise it returns `ok`.

## Create the functions

Choose a language and scaffold the four functions in a single `stack.yaml`
file:

=== "Go"

    ```bash
    faas-cli template store pull golang-middleware

    faas-cli new --lang golang-middleware telemetry-workflow \
      --prefix ttl.sh/openfaas-examples

    faas-cli new --lang golang-middleware validate-reading \
      --append stack.yaml --prefix ttl.sh/openfaas-examples

    faas-cli new --lang golang-middleware temperature-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples

    faas-cli new --lang golang-middleware battery-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

    Replace the generated handler files and `stack.yaml` with the Go files from
    the implementation section below.

=== "Python"

    ```bash
    faas-cli template store pull python3-http

    faas-cli new --lang python3-http telemetry-workflow \
      --prefix ttl.sh/openfaas-examples

    faas-cli new --lang python3-http validate-reading \
      --append stack.yaml --prefix ttl.sh/openfaas-examples

    faas-cli new --lang python3-http temperature-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples

    faas-cli new --lang python3-http battery-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

    Replace the generated handler files, `telemetry-workflow/requirements.txt`,
    and `stack.yaml` with the Python files from the implementation section
    below.

=== "Node.js"

    ```bash
    faas-cli template store pull node24

    faas-cli new --lang node24 telemetry-workflow \
      --prefix ttl.sh/openfaas-examples

    faas-cli new --lang node24 validate-reading \
      --append stack.yaml --prefix ttl.sh/openfaas-examples

    faas-cli new --lang node24 temperature-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples

    faas-cli new --lang node24 battery-check \
      --append stack.yaml --prefix ttl.sh/openfaas-examples
    ```

    Replace the generated handler files and `stack.yaml` with the Node.js files
    from the implementation section below.

The example uses the public [ttl.sh](https://ttl.sh) registry. Replace the
prefix with your own registry for production use.

## Implement the workflow

### Director: telemetry-workflow

=== "Go"

    `telemetry-workflow/handler.go`:

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

    const defaultStageTimeout = 5 * time.Second

    type Reading struct {
        DeviceID       string  `json:"device_id"`
        TemperatureC   float64 `json:"temperature_c"`
        BatteryPercent int     `json:"battery_percent"`
    }

    type TemperatureResult struct {
        ValueC     float64 `json:"value_c"`
        ThresholdC float64 `json:"threshold_c"`
        Alert      bool    `json:"alert"`
    }

    type BatteryResult struct {
        ValuePercent     int  `json:"value_percent"`
        ThresholdPercent int  `json:"threshold_percent"`
        Alert            bool `json:"alert"`
    }

    type Response struct {
        DeviceID    string            `json:"device_id"`
        Status      string            `json:"status"`
        Temperature TemperatureResult `json:"temperature"`
        Battery     BatteryResult     `json:"battery"`
        DurationMs  int64             `json:"duration_ms"`
    }

    type callResult struct {
        function string
        body     []byte
        status   int
        err      error
    }

    func Handle(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        input, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, "unable to read request body", http.StatusBadRequest)
            return
        }
        defer r.Body.Close()

        timeout, err := configuredStageTimeout()
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }

        gateway := os.Getenv("gateway_url")
        if gateway == "" {
            gateway = "http://gateway.openfaas:8080"
        }

        client := &http.Client{Timeout: timeout}

        validated, status, err := invoke(
            r.Context(), client, gateway, "validate-reading", input,
        )
        if err != nil {
            message := fmt.Sprintf("failed to invoke validate-reading: %s", err)
            http.Error(w, message, http.StatusBadGateway)
            return
        }
        if status != http.StatusOK {
            message := fmt.Sprintf("validate-reading failed: %s", validated)
            http.Error(w, message, status)
            return
        }

        var reading Reading
        if err := json.Unmarshal(validated, &reading); err != nil {
            message := fmt.Sprintf(
                "unexpected response from validate-reading: %s",
                err,
            )
            http.Error(w, message, http.StatusBadGateway)
            return
        }

        functions := []string{"temperature-check", "battery-check"}
        results := make(chan callResult, len(functions))

        for _, function := range functions {
            go func(name string) {
                body, status, err := invoke(
                    r.Context(), client, gateway, name, validated,
                )
                results <- callResult{
                    function: name,
                    body:     body,
                    status:   status,
                    err:      err,
                }
            }(function)
        }

        completed := make(map[string]callResult, len(functions))
        for range functions {
            result := <-results
            completed[result.function] = result
        }

        for _, function := range functions {
            result := completed[function]
            if result.err != nil {
                message := fmt.Sprintf(
                    "failed to invoke %s: %s",
                    function,
                    result.err,
                )
                http.Error(w, message, http.StatusBadGateway)
                return
            }
            if result.status != http.StatusOK {
                message := fmt.Sprintf("%s failed: %s", function, result.body)
                http.Error(w, message, result.status)
                return
            }
        }

        var temperature TemperatureResult
        if err := json.Unmarshal(
            completed["temperature-check"].body,
            &temperature,
        ); err != nil {
            message := fmt.Sprintf(
                "unexpected response from temperature-check: %s",
                err,
            )
            http.Error(w, message, http.StatusBadGateway)
            return
        }

        var battery BatteryResult
        if err := json.Unmarshal(
            completed["battery-check"].body,
            &battery,
        ); err != nil {
            message := fmt.Sprintf(
                "unexpected response from battery-check: %s",
                err,
            )
            http.Error(w, message, http.StatusBadGateway)
            return
        }

        workflowStatus := "ok"
        if temperature.Alert || battery.Alert {
            workflowStatus = "alert"
        }

        response := Response{
            DeviceID:    reading.DeviceID,
            Status:      workflowStatus,
            Temperature: temperature,
            Battery:     battery,
            DurationMs:  time.Since(start).Milliseconds(),
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(response)
    }

    func configuredStageTimeout() (time.Duration, error) {
        value := os.Getenv("stage_timeout")
        if value == "" {
            return defaultStageTimeout, nil
        }

        timeout, err := time.ParseDuration(value)
        if err != nil || timeout <= 0 {
            return 0, fmt.Errorf("invalid stage_timeout %q", value)
        }

        return timeout, nil
    }

    func invoke(
        ctx context.Context,
        client *http.Client,
        gateway string,
        function string,
        body []byte,
    ) ([]byte, int, error) {
        url := strings.TrimRight(gateway, "/") + "/function/" + function
        req, err := http.NewRequestWithContext(
            ctx,
            http.MethodPost,
            url,
            bytes.NewReader(body),
        )
        if err != nil {
            return nil, 0, fmt.Errorf("create request for %s: %w", function, err)
        }
        req.Header.Set("Content-Type", "application/json")

        res, err := client.Do(req)
        if err != nil {
            return nil, 0, fmt.Errorf("invoke %s: %w", function, err)
        }
        defer res.Body.Close()

        out, err := io.ReadAll(res.Body)
        if err != nil {
            return nil, 0, fmt.Errorf("read response from %s: %w", function, err)
        }

        return out, res.StatusCode, nil
    }
    ```

=== "Python"

    `telemetry-workflow/handler.py`:

    ```python
    import concurrent.futures
    import os
    import time

    import requests


    GATEWAY_URL = os.getenv(
        "gateway_url", "http://gateway.openfaas:8080"
    ).rstrip("/")
    STAGE_TIMEOUT = float(os.getenv("stage_timeout", "5"))
    if STAGE_TIMEOUT <= 0:
        raise ValueError("stage_timeout must be greater than zero")


    def handle(event, context):
        started = time.monotonic()
        body = event.body

        try:
            validated = invoke("validate-reading", body)
        except requests.RequestException as err:
            return error(502, f"failed to invoke validate-reading: {err}")

        if validated.status_code != 200:
            return error(
                validated.status_code,
                f"validate-reading failed: {validated.text}",
            )

        try:
            reading = validated.json()
        except ValueError as err:
            return error(502, f"unexpected response from validate-reading: {err}")

        functions = ("temperature-check", "battery-check")
        completed = {}
        with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
            futures = {
                name: executor.submit(invoke, name, validated.content)
                for name in functions
            }
            for name, future in futures.items():
                try:
                    completed[name] = future.result()
                except requests.RequestException as err:
                    return error(502, f"failed to invoke {name}: {err}")

        for name in functions:
            response = completed[name]
            if response.status_code != 200:
                return error(
                    response.status_code,
                    f"{name} failed: {response.text}",
                )

        try:
            temperature = completed["temperature-check"].json()
            battery = completed["battery-check"].json()
        except ValueError as err:
            return error(502, f"unexpected response from check function: {err}")

        has_alert = temperature["alert"] or battery["alert"]
        workflow_status = "alert" if has_alert else "ok"
        return {
            "statusCode": 200,
            "body": {
                "device_id": reading["device_id"],
                "status": workflow_status,
                "temperature": temperature,
                "battery": battery,
                "duration_ms": int((time.monotonic() - started) * 1000),
            },
        }


    def invoke(function, body):
        return requests.post(
            f"{GATEWAY_URL}/function/{function}",
            data=body,
            headers={"Content-Type": "application/json"},
            timeout=STAGE_TIMEOUT,
        )


    def error(status_code, message):
        return {"statusCode": status_code, "body": message.strip()}
    ```

    `telemetry-workflow/requirements.txt`:

    ```text
    requests
    ```

=== "Node.js"

    `telemetry-workflow/handler.js`:

    ```javascript
    'use strict'

    const { performance } = require('node:perf_hooks')

    const gatewayURL = process.env.gateway_url ||
      'http://gateway.openfaas:8080'
    const stageTimeout = configuredTimeout(
      process.env.stage_timeout || '5',
      'stage_timeout'
    )

    module.exports = async (event, context) => {
      const started = performance.now()
      const input = requestBody(event.body)

      let validated
      try {
        validated = await invoke('validate-reading', input)
      } catch (error) {
        return fail(
          context,
          502,
          `failed to invoke validate-reading: ${error.message}`
        )
      }

      if (validated.status !== 200) {
        return fail(
          context,
          validated.status,
          `validate-reading failed: ${await validated.text()}`
        )
      }

      let reading
      try {
        reading = await validated.json()
      } catch (error) {
        return fail(
          context,
          502,
          `unexpected response from validate-reading: ${error.message}`
        )
      }

      const body = JSON.stringify(reading)
      const names = ['temperature-check', 'battery-check']
      let responses
      try {
        responses = await Promise.all(
          names.map(async (name) => [name, await invoke(name, body)])
        )
      } catch (error) {
        return fail(context, 502, `failed to invoke check: ${error.message}`)
      }

      const completed = Object.fromEntries(responses)
      for (const name of names) {
        const response = completed[name]
        if (response.status !== 200) {
          return fail(
            context,
            response.status,
            `${name} failed: ${await response.text()}`
          )
        }
      }

      let temperature
      let battery
      try {
        temperature = await completed['temperature-check'].json()
        battery = await completed['battery-check'].json()
      } catch (error) {
        return fail(
          context,
          502,
          `unexpected response from check function: ${error.message}`
        )
      }

      const status = temperature.alert || battery.alert ? 'alert' : 'ok'
      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed({
          device_id: reading.device_id,
          status,
          temperature,
          battery,
          duration_ms: Math.round(performance.now() - started)
        })
    }

    function invoke (name, body) {
      const gateway = gatewayURL.replace(/\/$/, '')
      return fetch(`${gateway}/function/${name}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body,
        signal: AbortSignal.timeout(stageTimeout)
      })
    }

    function requestBody (body) {
      if (Buffer.isBuffer(body)) {
        return body
      }
      return typeof body === 'string' ? body : JSON.stringify(body)
    }

    function configuredTimeout (value, name) {
      const seconds = Number(value)
      if (!Number.isFinite(seconds) || seconds <= 0) {
        throw new Error(`${name} must be greater than zero`)
      }
      return seconds * 1000
    }

    function fail (context, status, message) {
      return context
        .status(status)
        .headers({ 'Content-Type': 'text/plain' })
        .succeed(message.trim())
    }
    ```

The director invokes every stage through the gateway, so each stage can be
written in a different language and scaled independently. A stage transport
error returns `502 Bad Gateway`. A non-200 response is attributed to the stage
that returned it and passed through to the caller.

The two checks are started concurrently, and the director waits for both
before choosing the final workflow status.

### Stage: validate-reading

The validation stage normalizes the device ID and rejects invalid values before
the parallel checks consume capacity.

=== "Go"

    `validate-reading/handler.go`:

    ```go
    package function

    import (
        "encoding/json"
        "net/http"
        "strings"
    )

    type Reading struct {
        DeviceID       string  `json:"device_id"`
        TemperatureC   float64 `json:"temperature_c"`
        BatteryPercent int     `json:"battery_percent"`
    }

    func Handle(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        var reading Reading
        if err := json.NewDecoder(r.Body).Decode(&reading); err != nil {
            http.Error(w, "expected a JSON sensor reading", http.StatusBadRequest)
            return
        }

        reading.DeviceID = strings.TrimSpace(reading.DeviceID)
        if reading.DeviceID == "" {
            http.Error(w, "device_id is required", http.StatusBadRequest)
            return
        }
        if reading.TemperatureC < -100 || reading.TemperatureC > 200 {
            http.Error(
                w,
                "temperature_c must be between -100 and 200",
                http.StatusBadRequest,
            )
            return
        }
        if reading.BatteryPercent < 0 || reading.BatteryPercent > 100 {
            http.Error(
                w,
                "battery_percent must be between 0 and 100",
                http.StatusBadRequest,
            )
            return
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(reading)
    }
    ```

=== "Python"

    `validate-reading/handler.py`:

    ```python
    import json


    def handle(event, context):
        try:
            reading = json.loads(event.body)
        except (TypeError, ValueError):
            return error("expected a JSON sensor reading")

        device_id = str(reading.get("device_id", "")).strip()
        temperature = reading.get("temperature_c", 0)
        battery = reading.get("battery_percent", 0)

        if not device_id:
            return error("device_id is required")
        if not is_number(temperature) or temperature < -100 or temperature > 200:
            return error("temperature_c must be between -100 and 200")
        if (
            not isinstance(battery, int)
            or isinstance(battery, bool)
            or battery < 0
            or battery > 100
        ):
            return error("battery_percent must be between 0 and 100")

        return {
            "statusCode": 200,
            "body": {
                "device_id": device_id,
                "temperature_c": temperature,
                "battery_percent": battery,
            },
        }


    def is_number(value):
        return isinstance(value, (int, float)) and not isinstance(value, bool)


    def error(message):
        return {"statusCode": 400, "body": message}
    ```

=== "Node.js"

    `validate-reading/handler.js`:

    ```javascript
    'use strict'

    module.exports = async (event, context) => {
      let reading
      try {
        reading = parseBody(event.body)
      } catch (error) {
        return fail(context, 'expected a JSON sensor reading')
      }

      const deviceID = String(reading.device_id || '').trim()
      const temperature = reading.temperature_c ?? 0
      const battery = reading.battery_percent ?? 0

      if (!deviceID) {
        return fail(context, 'device_id is required')
      }
      if (
        !Number.isFinite(temperature) ||
        temperature < -100 ||
        temperature > 200
      ) {
        return fail(
          context,
          'temperature_c must be between -100 and 200'
        )
      }
      if (!Number.isInteger(battery) || battery < 0 || battery > 100) {
        return fail(
          context,
          'battery_percent must be between 0 and 100'
        )
      }

      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed({
          device_id: deviceID,
          temperature_c: temperature,
          battery_percent: battery
        })
    }

    function parseBody (body) {
      if (Buffer.isBuffer(body)) {
        return JSON.parse(body.toString())
      }
      return typeof body === 'string' ? JSON.parse(body) : body
    }

    function fail (context, message) {
      return context
        .status(400)
        .headers({ 'Content-Type': 'text/plain' })
        .succeed(message)
    }
    ```

### Parallel stages: temperature-check and battery-check

The checks are intentionally small. In a real workflow they could call a model,
query device metadata, or apply rules maintained by another team.

**Temperature check**

=== "Go"

    `temperature-check/handler.go`:

    ```go
    package function

    import (
        "encoding/json"
        "net/http"
    )

    const thresholdC = 75.0

    type Reading struct {
        TemperatureC float64 `json:"temperature_c"`
    }

    type Response struct {
        ValueC     float64 `json:"value_c"`
        ThresholdC float64 `json:"threshold_c"`
        Alert      bool    `json:"alert"`
    }

    func Handle(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        var reading Reading
        if err := json.NewDecoder(r.Body).Decode(&reading); err != nil {
            http.Error(w, "expected a JSON sensor reading", http.StatusBadRequest)
            return
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(Response{
            ValueC:     reading.TemperatureC,
            ThresholdC: thresholdC,
            Alert:      reading.TemperatureC > thresholdC,
        })
    }
    ```

=== "Python"

    `temperature-check/handler.py`:

    ```python
    import json


    THRESHOLD_C = 75.0


    def handle(event, context):
        try:
            reading = json.loads(event.body)
            value = reading["temperature_c"]
        except (KeyError, TypeError, ValueError):
            return {"statusCode": 400, "body": "expected a JSON sensor reading"}

        return {
            "statusCode": 200,
            "body": {
                "value_c": value,
                "threshold_c": THRESHOLD_C,
                "alert": value > THRESHOLD_C,
            },
        }
    ```

=== "Node.js"

    `temperature-check/handler.js`:

    ```javascript
    'use strict'

    const thresholdC = 75.0

    module.exports = async (event, context) => {
      let reading
      try {
        reading = parseBody(event.body)
      } catch (error) {
        return fail(context)
      }

      const value = reading.temperature_c
      if (!Number.isFinite(value)) {
        return fail(context)
      }

      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed({
          value_c: value,
          threshold_c: thresholdC,
          alert: value > thresholdC
        })
    }

    function parseBody (body) {
      if (Buffer.isBuffer(body)) {
        return JSON.parse(body.toString())
      }
      return typeof body === 'string' ? JSON.parse(body) : body
    }

    function fail (context) {
      return context
        .status(400)
        .headers({ 'Content-Type': 'text/plain' })
        .succeed('expected a JSON sensor reading')
    }
    ```

**Battery check**

=== "Go"

    `battery-check/handler.go`:

    ```go
    package function

    import (
        "encoding/json"
        "net/http"
    )

    const thresholdPercent = 20

    type Reading struct {
        BatteryPercent int `json:"battery_percent"`
    }

    type Response struct {
        ValuePercent     int  `json:"value_percent"`
        ThresholdPercent int  `json:"threshold_percent"`
        Alert            bool `json:"alert"`
    }

    func Handle(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        var reading Reading
        if err := json.NewDecoder(r.Body).Decode(&reading); err != nil {
            http.Error(w, "expected a JSON sensor reading", http.StatusBadRequest)
            return
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(Response{
            ValuePercent:     reading.BatteryPercent,
            ThresholdPercent: thresholdPercent,
            Alert:            reading.BatteryPercent < thresholdPercent,
        })
    }
    ```

=== "Python"

    `battery-check/handler.py`:

    ```python
    import json


    THRESHOLD_PERCENT = 20


    def handle(event, context):
        try:
            reading = json.loads(event.body)
            value = reading["battery_percent"]
        except (KeyError, TypeError, ValueError):
            return {"statusCode": 400, "body": "expected a JSON sensor reading"}

        return {
            "statusCode": 200,
            "body": {
                "value_percent": value,
                "threshold_percent": THRESHOLD_PERCENT,
                "alert": value < THRESHOLD_PERCENT,
            },
        }
    ```

=== "Node.js"

    `battery-check/handler.js`:

    ```javascript
    'use strict'

    const thresholdPercent = 20

    module.exports = async (event, context) => {
      let reading
      try {
        reading = parseBody(event.body)
      } catch (error) {
        return fail(context)
      }

      const value = reading.battery_percent
      if (!Number.isInteger(value)) {
        return fail(context)
      }

      return context
        .status(200)
        .headers({ 'Content-Type': 'application/json' })
        .succeed({
          value_percent: value,
          threshold_percent: thresholdPercent,
          alert: value < thresholdPercent
        })
    }

    function parseBody (body) {
      if (Buffer.isBuffer(body)) {
        return JSON.parse(body.toString())
      }
      return typeof body === 'string' ? JSON.parse(body) : body
    }

    function fail (context) {
      return context
        .status(400)
        .headers({ 'Content-Type': 'text/plain' })
        .succeed('expected a JSON sensor reading')
    }
    ```

### Stack file

=== "Go"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      telemetry-workflow:
        lang: golang-middleware
        handler: ./telemetry-workflow
        image: ttl.sh/openfaas-examples/telemetry-workflow:latest
        environment:
          stage_timeout: 5s
          exec_timeout: 15s
          read_timeout: 16s
          write_timeout: 16s

      validate-reading:
        lang: golang-middleware
        handler: ./validate-reading
        image: ttl.sh/openfaas-examples/validate-reading:latest

      temperature-check:
        lang: golang-middleware
        handler: ./temperature-check
        image: ttl.sh/openfaas-examples/temperature-check:latest

      battery-check:
        lang: golang-middleware
        handler: ./battery-check
        image: ttl.sh/openfaas-examples/battery-check:latest
    ```

=== "Python"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      telemetry-workflow:
        lang: python3-http
        handler: ./telemetry-workflow
        image: ttl.sh/openfaas-examples/python-telemetry-workflow:latest
        build_args:
          TEST_ENABLED: "true"
        environment:
          stage_timeout: "5"
          exec_timeout: 15s
          read_timeout: 16s
          write_timeout: 16s

      validate-reading:
        lang: python3-http
        handler: ./validate-reading
        image: ttl.sh/openfaas-examples/python-validate-reading:latest
        build_args:
          TEST_ENABLED: "true"

      temperature-check:
        lang: python3-http
        handler: ./temperature-check
        image: ttl.sh/openfaas-examples/python-temperature-check:latest
        build_args:
          TEST_ENABLED: "true"

      battery-check:
        lang: python3-http
        handler: ./battery-check
        image: ttl.sh/openfaas-examples/python-battery-check:latest
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
      telemetry-workflow:
        lang: node24
        handler: ./telemetry-workflow
        image: ttl.sh/openfaas-examples/node-telemetry-workflow:latest
        environment:
          stage_timeout: "5"
          exec_timeout: 15s
          read_timeout: 16s
          write_timeout: 16s

      validate-reading:
        lang: node24
        handler: ./validate-reading
        image: ttl.sh/openfaas-examples/node-validate-reading:latest

      temperature-check:
        lang: node24
        handler: ./temperature-check
        image: ttl.sh/openfaas-examples/node-temperature-check:latest

      battery-check:
        lang: node24
        handler: ./battery-check
        image: ttl.sh/openfaas-examples/node-battery-check:latest
    ```

## Configure timeouts

A director stays active while it waits for the stages it invokes. This example
uses two kinds of timeout:

| Setting | Scope |
|---------|-------|
| `stage_timeout` | Application-level HTTP client timeout for each downstream call |
| `exec_timeout` | Maximum duration of the complete director invocation |
| `read_timeout`, `write_timeout` | Watchdog HTTP timeouts, set slightly longer than `exec_timeout` |

The director calls `validate-reading` first, followed by the two checks in
parallel. Its expected duration is therefore the validation duration plus the
slower of the two checks, with some additional overhead. Configure the
director's timeout for that complete path, not for a single stage.

The gateway's `upstream_timeout` must be at least as long as the director's
`exec_timeout`. Adjust the example values for your own functions and see
[extended timeouts](/tutorials/expanded-timeouts/) for the complete
configuration.

## Deploy and invoke

Build, push, and deploy all four functions:

```bash
faas-cli up --tag=sha
```

Invoke the director with a reading that exceeds both thresholds:

```bash
curl -s http://127.0.0.1:8080/function/telemetry-workflow \
  -H "Content-Type: application/json" \
  -d '{"device_id":"pump-17","temperature_c":82.4,"battery_percent":12}' | \
  jq
```

The director combines the two check results and selects the `alert` path:

```json
{
  "device_id": "pump-17",
  "status": "alert",
  "temperature": {
    "value_c": 82.4,
    "threshold_c": 75,
    "alert": true
  },
  "battery": {
    "value_percent": 12,
    "threshold_percent": 20,
    "alert": true
  },
  "duration_ms": 4
}
```

Submit values within both thresholds to select the `ok` path:

```bash
curl -s http://127.0.0.1:8080/function/telemetry-workflow \
  -H "Content-Type: application/json" \
  -d '{"device_id":"pump-17","temperature_c":48.2,"battery_percent":78}' | \
  jq
```

## Workflow considerations

* A validation failure stops the workflow before either parallel check is
  invoked.
* Parallel stages should be independent. If one stage needs the result of
  another, invoke them in sequence instead.
* The final `ok` or `alert` decision is a small conditional branch. It could be
  extended to invoke a notification function for alerts or a storage function
  for normal readings.
* For a long-running workflow, invoke the director through
  `/async-function/telemetry-workflow` with an `X-Callback-Url`. The director
  continues to wait for its stages, while the client receives the final result
  through the callback. See
  [asynchronous functions](/reference/async/#how-it-works).
