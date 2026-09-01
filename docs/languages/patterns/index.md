OpenFaaS functions can be combined into workflows, run in the background or in
parallel, or use a fixed replica count. This page introduces common patterns
that work with any language template.

| Pattern | Summary | Useful when | Example |
|---------|-----------|-------------|---------|
| [Director](#director-pattern) | One function coordinates several other functions | A workflow needs sequencing, branching, parallel function calls, or a combined response | [View example](/languages/patterns/director/) |
| [Async invocation](#async-invocation-pattern) | The gateway queues an invocation for async processing | Work should continue in the background, or queued processing is needed for back pressure | — |
| [Fan-out](#fan-out-pattern) | One invocation is divided into many independent asynchronous invocations | A batch needs concurrent processing that can scale across multiple function replicas | [View example](/languages/patterns/fan-out/) |
| [Singleton](#singleton-pattern) | A function is kept at one replica | State is tied to a connection, or the function must be limited to a single replica | [View example](/languages/patterns/singleton/) |

The patterns can also be combined. For example, a director can fan out a batch
to another function, or an asynchronous invocation can start a director without
keeping the original caller connected.

## Director pattern

A director is an OpenFaaS function that coordinates a workflow. It invokes
other functions through the gateway, passes results between them, decides what
runs next, and returns or stores the final result.

The functions invoked by the director are deployed independently, so each can
use a different language, scale separately, and be updated without moving the
workflow logic out of the director.

```text
      [ Client ]
            │
            ▼ HTTP POST / invoke
┌──────────────────────┐
│  Director Function   │ ◄── Owns the workflow
└──────────┬───────────┘
           │
           ├── 1. Invoke ──► [ Function A ]
           │                        │
           │◄──── response ─────────┘
           │
           ├── 2. Branch on result
           │       ├── if X ──► [ Function B ]
           │       └── if Y ──► [ Function C ]
           │
           ├── 3. Invoke independent functions in parallel
           │       ├──► [ Function D ] ──┐
           │       └──► [ Function E ] ──┤
           │                             ▼
           │                     [ Combine results ]
           │
           ▼
[ Return response / persist result ]
```

**Useful when:**

* Several independently deployed functions form one logical operation.
* The caller needs one endpoint and one combined response.
* A workflow needs sequencing, branching, or independent functions to run
  in parallel.

**Design considerations:**

* The director remains active while it waits for downstream functions to
  finish. Its [timeouts](/tutorials/expanded-timeouts/) should be configured to
  cover the longest expected path through the complete workflow, rather than a
  single downstream function call.
* The director decides how to handle errors from the functions it invokes. It
  can stop, retry, return a partial result, or save its progress.
* For a long-running workflow, the director can be
  [invoked asynchronously](#async-invocation-pattern), so the caller does not
  need to wait. The director can send the final result to a callback URL when
  the workflow completes.

**Examples:**

* [Coordinate a workflow with a director](/languages/patterns/director/)

## Async invocation pattern

Any OpenFaaS function can be invoked asynchronously by changing the gateway
route from `/function/<name>` to `/async-function/<name>`. The caller sends an
HTTP POST and immediately receives `202 Accepted`. The request is added to a
queue and processed later by the queue-worker.

This decouples the HTTP request from the function response, so the caller does
not need to wait while long-running work, background jobs, or batches are
processed.

An optional callback can receive the result when processing
completes.

```text
 [ Client ] ──► [ Gateway ] ──► [ Queue ]
      ▲              │              │
      └──── 202 ─────┘              ▼
                               [ Queue-worker ] ──► [ Function ]
                                    │
                                    └── callback ──► [ Result endpoint ]
```

**Useful when:**

* Work takes longer than the caller should wait.
* Queued processing provides back pressure when requests arrive faster than
  available capacity.
* Failed asynchronous invocations need to be
  [retried automatically](/openfaas-pro/retries/) with a back-off.

See [asynchronous functions](/reference/async/) for more information on
invocations, callbacks, retries and queue configuration.

## Fan-out pattern

Fan-out combines asynchronous invocation with independently scalable
functions. One function divides a batch into independent items and invokes a
worker function asynchronously for each item. It returns after the invocations
are queued rather than waiting for them to finish.

The queue-worker drains the queued items as capacity becomes available, while
OpenFaaS can [autoscale the target function](/architecture/autoscaling/) across
multiple replicas.

The queue-worker automatically
[retries failed invocations](/openfaas-pro/retries/), allowing batch items to
complete even when an invocation fails.

```text
       [ Input batch ]
              │
              ▼
┌────────────────────────┐
│    Fan-out Function    │
└────────────┬───────────┘
             │
             ▼
┌────────────────────────┐
│      Queue-worker      │
└────────────┬───────────┘
             ├── item 1 / async ──► [ Function ] ──┐
             ├── item 2 / async ──► [ Function ] ──┤
             └── item N / async ──► [ Function ] ──┘
                                                  │ callback
                                                  ▼
                                         [ Result collector ]
```

**Useful when:**

* A batch contains records that can be processed independently.
* Processing should happen concurrently and can complete out of order.
* Processing capacity needs to scale across multiple worker function replicas,
  independently of the function that submits the batch.

**Design considerations:**

* Items must be independent. If one item needs another item's result, process
  them in sequence instead.
* Callbacks can deliver each result independently and may arrive in a different
  order from the input.

**Fan-in:**

When the next step depends on every item completing, the individual
results can be fanned back in. One way to implement this is to:

* Set a counter to the number of items in the batch.
* Store each callback result in shared storage and atomically decrement the
  counter.
* When the counter reaches zero, invoke a final function to combine the stored
  results, send a notification, or start the next step in the workflow.

When results do not need to be combined, each callback can be handled
independently.

**Examples:**

* [Process a batch with fan-out](/languages/patterns/fan-out/)
* [Combine individual results with fan-out and fan-in](https://www.openfaas.com/blog/fan-out-and-back-in-using-functions/).

## Singleton pattern

A singleton function is configured with a fixed replica count of one. It can be
useful when a workload keeps connection-local state, wraps software that cannot
run concurrently, or must limit access to an external resource.

```text
        [ Requests ]
             │
             ▼ HTTP / invoke
┌──────────────────────────┐
│    Singleton Function    │ ◄── Fixed at one replica
│                          │
│       [ Replica 1 ]      │
└──────────────────────────┘
```

Set the minimum and maximum replica labels to the same value to disable
horizontal scaling for the function:

```yaml
functions:
  stateful-function:
    labels:
      com.openfaas.scale.min: 1
      com.openfaas.scale.max: 1
```

**Useful when:**

* Connections or subscribers are stored in the function process.
* Software or an external resource requires a fixed number of function
  replicas.
* Horizontal scaling is intentionally undesirable.

**Design considerations:**

* A fixed replica count keeps one replica running during normal operation, but
  does not make the function or its local filesystem durable. The replica can
  be replaced during a restart, rescheduling, or deployment, so state that
  needs to survive these events should still be stored externally.
* The function cannot add replicas to increase its capacity.
* If the goal is to process one request at a time, set the `max_inflight: 1`
  environment variable on the function. Fixing the replica count alone does
  not prevent concurrent requests within the replica.

For more information on configuring function scaling, see the
[autoscaling documentation](/architecture/autoscaling/).

**Examples:**

* [Run a function as a singleton](/languages/patterns/singleton/)
* [Apply singleton scaling to WebSocket connections](https://www.openfaas.com/blog/serverless-websockets/#scaling-websockets).
