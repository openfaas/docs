The [Singleton pattern](/languages/patterns/#singleton-pattern) fixes a
function's desired replica count at one. This example demonstrates the pattern
with a small Server-Sent Events (SSE) notification hub. Singleton functions are
useful when a workload keeps connection-local state or wraps software that
cannot run concurrently.

Use-cases:

* Keeping connection-local state, such as SSE subscribers or WebSocket sessions
* Wrapping software that cannot safely run concurrently
* Limiting access to an external resource that permits one active client

## How it works

The `notification-hub` function accepts two kinds of request:

```text
[ Publisher ] ── POST ──► [ notification-hub ] ── SSE ──► [ Subscribers ]
                              one replica
```

`GET` opens an SSE subscription, while `POST` broadcasts its request body to
all connected subscribers.

The function runs with one replica, so every subscription is registered in the
same in-memory map and each notification can reach all connected subscribers.

## Create the function

Choose a language and scaffold the function:

=== "Go"

    ```bash
    faas-cli template store pull golang-middleware

    faas-cli new --lang golang-middleware notification-hub \
      --prefix ttl.sh/openfaas-examples
    ```

=== "Python"

    ```bash
    faas-cli template store pull python3-flask

    faas-cli new --lang python3-flask notification-hub \
      --prefix ttl.sh/openfaas-examples
    ```

Replace the generated handler and `stack.yaml` with the files from the
implementation section below.

The example uses the public [ttl.sh](https://ttl.sh) registry. Replace the
prefix with your own registry for production use.

## Implement the function

=== "Go"

    `handler.go`:

    ```go
    package function

    import (
        "fmt"
        "io"
        "net/http"
        "strings"
        "sync"
    )

    var (
        subscribersMu sync.Mutex
        subscribers   = make(map[chan string]struct{})
    )

    func Handle(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodGet:
            subscribe(w, r)
        case http.MethodPost:
            publish(w, r)
        default:
            w.Header().Set("Allow", "GET, POST")
            http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
        }
    }

    func subscribe(w http.ResponseWriter, r *http.Request) {
        flusher, ok := w.(http.Flusher)
        if !ok {
            http.Error(
                w,
                "streaming is not supported",
                http.StatusInternalServerError,
            )
            return
        }

        messages := make(chan string, 1)
        subscribersMu.Lock()
        subscribers[messages] = struct{}{}
        subscribersMu.Unlock()

        defer func() {
            subscribersMu.Lock()
            delete(subscribers, messages)
            subscribersMu.Unlock()
        }()

        w.Header().Set("Content-Type", "text/event-stream")
        w.Header().Set("Cache-Control", "no-cache")
        fmt.Fprint(w, ": connected\n\n")
        flusher.Flush()

        for {
            select {
            case message := <-messages:
                fmt.Fprintf(w, "data: %s\n\n", message)
                flusher.Flush()
            case <-r.Context().Done():
                return
            }
        }
    }

    func publish(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        body, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, "unable to read request body", http.StatusBadRequest)
            return
        }

        message := strings.TrimSpace(string(body))
        if message == "" {
            http.Error(w, "notification must not be empty", http.StatusBadRequest)
            return
        }

        message = strings.NewReplacer("\r", " ", "\n", " ").Replace(message)
        delivered := broadcast(message)

        w.Header().Set("Content-Type", "application/json")
        fmt.Fprintf(w, "{\"delivered\":%d}\n", delivered)
    }

    func broadcast(message string) int {
        subscribersMu.Lock()
        defer subscribersMu.Unlock()

        delivered := 0
        for subscriber := range subscribers {
            select {
            case subscriber <- message:
                delivered++
            default:
                // Do not let a slow subscriber block every other client.
            }
        }

        return delivered
    }
    ```

=== "Python"

    `handler.py`:

    ```python
    import json
    import queue
    import threading

    from flask import Response, request


    subscribers = set()
    subscribers_lock = threading.Lock()
    HEARTBEAT_INTERVAL = 15


    def handle(req):
        if request.method == "GET":
            return subscribe()
        if request.method == "POST":
            return publish(req)

        return "method not allowed", 405, {"Allow": "GET, POST"}


    def subscribe():
        messages = queue.Queue(maxsize=1)
        with subscribers_lock:
            subscribers.add(messages)

        def stream():
            try:
                yield ": connected\n\n"
                while True:
                    try:
                        message = messages.get(timeout=HEARTBEAT_INTERVAL)
                        yield f"data: {message}\n\n"
                    except queue.Empty:
                        # Periodic writes let the server detect idle disconnects.
                        yield ": keep-alive\n\n"
            finally:
                with subscribers_lock:
                    subscribers.discard(messages)

        return Response(
            stream(),
            mimetype="text/event-stream",
            headers={"Cache-Control": "no-cache"},
        )


    def publish(req):
        body = req.decode() if isinstance(req, bytes) else str(req)
        message = " ".join(body.splitlines()).strip()
        if not message:
            return "notification must not be empty", 400

        delivered = broadcast(message)
        return (
            json.dumps({"delivered": delivered}) + "\n",
            200,
            {"Content-Type": "application/json"},
        )


    def broadcast(message):
        delivered = 0
        with subscribers_lock:
            for subscriber in subscribers:
                try:
                    subscriber.put_nowait(message)
                    delivered += 1
                except queue.Full:
                    # Do not let a slow subscriber block every other client.
                    pass

        return delivered
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
      notification-hub:
        lang: golang-middleware
        handler: ./notification-hub
        image: ttl.sh/openfaas-examples/notification-hub:latest
        labels:
          com.openfaas.scale.min: "1"
          com.openfaas.scale.max: "1"
        environment:
          exec_timeout: "1h"
          read_timeout: "1h1s"
          write_timeout: "1h1s"
    ```

=== "Python"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      notification-hub:
        lang: python3-flask
        handler: ./notification-hub
        image: ttl.sh/openfaas-examples/python-notification-hub:latest
        build_args:
          TEST_ENABLED: "true"
        labels:
          com.openfaas.scale.min: "1"
          com.openfaas.scale.max: "1"
        environment:
          exec_timeout: "1h"
          read_timeout: "1h1s"
          write_timeout: "1h1s"
    ```

Setting both replica limits to one disables horizontal scaling and makes the
function a singleton. With multiple replicas, a notification would only reach
subscribers connected to the replica that received it. See
[autoscaling](/architecture/autoscaling/) for more detail.

Do not set `max_inflight` to one for this function. It limits how many requests
a replica can process concurrently. Each SSE subscription remains in flight,
so a limit of one would prevent publishers from connecting. See
[concurrent request limits](/architecture/invocations/#how-many-times-can-a-function-be-invoked)
for more information on limiting requests.

## Deploy and invoke

Build and deploy the function:

```bash
faas-cli up --tag=sha
```

Use `faas-cli list -v` to confirm that the function has one configured and
available replica:

```bash
faas-cli list -v
```

Open an SSE subscription in one terminal. The `Accept` header tells the gateway
to stream the response:

```bash
curl -N -H "Accept: text/event-stream" \
  http://127.0.0.1:8080/function/notification-hub
```

Publish a notification from another terminal:

```bash
curl -s -d "deployment complete" \
  http://127.0.0.1:8080/function/notification-hub | jq
```

The publisher reports how many subscribers accepted the notification:

```json
{
  "delivered": 1
}
```

The subscriber receives:

```text
data: deployment complete
```

## Lifecycle and production considerations

Fixing the desired replica count at one does not make that replica durable. The
subscriber map exists for the lifetime of the function process. A restart,
reschedule, or deployment creates a new process with an empty map, closes the
existing connections, and requires clients to reconnect.

WebSockets have the same connection-local state concern and add bidirectional
communication. See
[How to Integrate WebSockets with Serverless Functions and OpenFaaS](https://www.openfaas.com/blog/serverless-websockets/).

Long-lived connections also require suitable function, gateway, ingress, and
load-balancer timeouts. See [extended timeouts](/tutorials/expanded-timeouts/).

This example uses an in-memory subscriber map to demonstrate the singleton
pattern. For a real notification implementation, consider using a message bus
to distribute notifications across replicas, and a database or durable message
bus when clients need history or guaranteed delivery.
