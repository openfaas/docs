The [Singleton pattern](/languages/patterns/#singleton-pattern) configures a
function with a desired replica count of one. With one replica, every request
or connection is routed to the same function process, allowing them to share
process-local state and reuse the same resources.

This page uses a deliberately small example: a function performs simulated
expensive setup once when its process starts, then reuses the resulting
resource for every request.

```text
                                    ┌──────────────────────────┐
[ Client A ] ──┐                    │    Singleton Function    │
[ Client B ] ──┼── connections ────►│                          │
[ Client C ] ──┘                    │       [ Replica 1 ]      │
                                    └──────────────────────────┘
```

This simple function highlights several properties of the Singleton pattern:

* **One replica:** setting both replica limits to one disables horizontal
  scaling, so OpenFaaS does not add replicas as load increases.
* **Reuse:** requests are routed to the same replica, so they can use the same
  process-local resource.

The example uses a two-second delay to simulate expensive setup. In a real
function, the same point in the lifecycle could be used to:

* Create a database connection pool or an SDK client.
* Download and load a large dataset or machine-learning model.
* Open a persistent connection whose state remains in the function process.
* Establish a session with an upstream service that has costly setup or permits
  only one active client.

## Create the function

Choose a language and scaffold the function:

=== "Go"

    ```bash
    faas-cli template store pull golang-middleware

    faas-cli new --lang golang-middleware singleton \
      --prefix ttl.sh/openfaas-examples
    ```

=== "Python"

    ```bash
    faas-cli template store pull python3-http

    faas-cli new --lang python3-http singleton \
      --prefix ttl.sh/openfaas-examples
    ```

=== "Node.js"

    ```bash
    faas-cli template store pull node24

    faas-cli new --lang node24 singleton \
      --prefix ttl.sh/openfaas-examples
    ```

The example uses the public [ttl.sh](https://ttl.sh) registry. Replace the
prefix with your own registry for production use.

The full source code and `stack.yaml` files are available on GitHub for
[Go](https://github.com/openfaas/function-patterns/tree/master/go/singleton),
[Python](https://github.com/openfaas/function-patterns/tree/master/python/singleton),
and [Node.js](https://github.com/openfaas/function-patterns/tree/master/node/singleton).

## Implement the function

Each implementation starts the simulated setup when the function process
starts. The `/ready` path returns `200` only after the resource is available.
Normal requests then reuse the resource.

=== "Go"

    `handler.go`:

    ```go
    package function

    import (
        "fmt"
        "net/http"
        "time"
    )

    var expensiveResource string

    func init() {
        // Initialize once when the function process starts, not on every request.
        expensiveResource = setupExpensiveResource()
    }

    func setupExpensiveResource() string {
        // A real function could create a database client, open a persistent
        // connection, or download a large dataset here.
        time.Sleep(2 * time.Second)
        return "The expensive resource is ready"
    }

    func Handle(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/ready" {
            fmt.Fprintln(w, "Ready")
            return
        }

        w.Header().Set("Content-Type", "text/plain; charset=utf-8")
        fmt.Fprintln(w, expensiveResource)
    }
    ```

=== "Python"

    `handler.py`:

    ```python
    import time


    def setup_expensive_resource():
        # A real function could create a database client, open a persistent
        # connection, or download a large dataset here.
        time.sleep(2)
        return "The expensive resource is ready"


    # Initialize once when the function process starts, not on every request.
    expensive_resource = setup_expensive_resource()


    def handle(event, context):
        if event.path == "/ready":
            return {"statusCode": 200, "body": "Ready"}

        return {"statusCode": 200, "body": expensive_resource}
    ```

=== "Node.js"

    `handler.js`:

    ```javascript
    'use strict'

    let expensiveResource
    let resourceReady = false

    async function setupExpensiveResource () {
      // A real function could create a database client, open a persistent
      // connection, or download a large dataset here.
      await new Promise(resolve => setTimeout(resolve, 2000))
      return 'The expensive resource is ready'
    }

    async function initialize () {
      expensiveResource = await setupExpensiveResource()
      resourceReady = true
    }

    // Initialize once when the function process starts, not on every request.
    initialize()

    module.exports = async (event, context) => {
      // The readiness probe holds traffic until initialization has completed.
      if (event.path === '/ready') {
        return respond(context, resourceReady ? 200 : 503,
          resourceReady ? 'Ready' : 'Initializing')
      }

      if (!resourceReady) {
        return respond(context, 503, 'Initializing')
      }

      return respond(context, 200, expensiveResource)
    }

    function respond (context, status, body) {
      return context
        .status(status)
        .headers({ 'Content-Type': 'text/plain; charset=utf-8' })
        .succeed(body)
    }
    ```

## Configure the singleton and readiness

=== "Go"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      singleton:
        lang: golang-middleware
        handler: ./singleton
        image: ttl.sh/openfaas-examples/singleton:latest
        labels:
          com.openfaas.scale.min: "1"
          com.openfaas.scale.max: "1"
        annotations:
          com.openfaas.ready.http.path: /ready
          com.openfaas.ready.http.initialDelaySeconds: 1
          com.openfaas.ready.http.periodSeconds: 1
    ```

=== "Python"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      singleton:
        lang: python3-http
        handler: ./singleton
        image: ttl.sh/openfaas-examples/python-singleton:latest
        labels:
          com.openfaas.scale.min: "1"
          com.openfaas.scale.max: "1"
        annotations:
          com.openfaas.ready.http.path: /ready
          com.openfaas.ready.http.initialDelaySeconds: 1
          com.openfaas.ready.http.periodSeconds: 1
    ```

=== "Node.js"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      singleton:
        lang: node24
        handler: ./singleton
        image: ttl.sh/openfaas-examples/node-singleton:latest
        labels:
          com.openfaas.scale.min: "1"
          com.openfaas.scale.max: "1"
        annotations:
          com.openfaas.ready.http.path: /ready
          com.openfaas.ready.http.initialDelaySeconds: 1
          com.openfaas.ready.http.periodSeconds: 1
    ```

Setting both replica limits to one disables horizontal scaling. With multiple
replicas, every process would perform the setup and keep its own copy of the
resource. See [autoscaling](/architecture/autoscaling/) for more detail.

The readiness annotations direct the platform probe to `/ready`. They prevent
the replica from receiving traffic while initialization is still running.
For more information, see
[Health and readiness for functions](https://www.openfaas.com/blog/health-and-readiness-for-functions/)
and [custom HTTP health checks](/reference/workloads/#custom-http-health-checks).

## Deploy and invoke

Build and deploy the function:

```bash
faas-cli up --tag=sha
```

Confirm that the function is deployed and has one available replica:

```bash
faas-cli list -v
```

Invoke the function after it becomes ready:

```bash
curl http://127.0.0.1:8080/function/singleton
```

The function should return:

```text
The expensive resource is ready
```

## Singleton considerations

* Process-local resources follow the replica lifecycle. Initialization runs
  again after a deployment, restart, or rescheduling, and local state is lost
  when the replica is replaced. Store durable state in external persistent
  storage.
* A single replica can still process multiple requests concurrently, so shared
  resources must be safe for concurrent use. Set `max_inflight: "1"` to process
  one request at a time; it limits concurrency, not the replica count. See
  [concurrent request limits](/architecture/invocations/#how-many-times-can-a-function-be-invoked).
* Stateful connections such as WebSockets are another use of the Singleton
  pattern because their connection state remains in one process. See
  [How to Integrate WebSockets with Serverless Functions and OpenFaaS](https://www.openfaas.com/blog/serverless-websockets/).
