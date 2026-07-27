Some functions need to do work before they can serve traffic: load a machine-learning model, warm a cache, open a database connection pool, or download a dataset. Until that work finishes, the function is running but not yet able to answer requests.

A readiness check lets OpenFaaS hold traffic back until the function signals it is ready. Without one, the first request after a cold start or a scale-from-zero lands on a Pod that is up but still initialising, and fails with a 500 or a timeout.

Use-cases:

* Loading an ML model or embeddings at start-up
* Warming an in-memory cache
* Opening a database or SDK connection pool
* Downloading reference data before the first request

!!! info "OpenFaaS Pro feature"
    Custom readiness checks are configured with the `com.openfaas.ready.http.*` annotations, which are part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution. On Community Edition the readiness probe always uses `/_/health` and these annotations are ignored.

## Handler

Run the slow initialisation off the request path — in a background thread started at module load — and set a flag when it finishes. The handler returns `200` from `/ready` once the flag is set, and `503` while it is still initialising.

handler.py:

```python
import threading
import time

# Ready is False until the background initialisation has finished.
ready = False


def initialize():
    global ready
    # Replace this with the real work: load a model, open a pool, warm a cache.
    time.sleep(10)
    ready = True


# Start the initialisation once, at module load, not on the request path.
threading.Thread(target=initialize, daemon=True).start()


def handle(event, context):
    if event.path == "/ready":
        if ready:
            return {"statusCode": 200, "body": "Ready"}
        return {"statusCode": 503, "body": "Initializing"}

    return {"statusCode": 200, "body": "Hello from OpenFaaS"}
```

The flag is a plain boolean, which is safe here: the background thread only ever sets it to `True`, and the handler only reads it.

## stack.yaml

Point the readiness probe at the `/ready` path with an annotation:

```yaml
functions:
  ready-example:
    lang: python3-http
    handler: ./ready-example
    image: ${REGISTRY}/ready-example:latest
    annotations:
      com.openfaas.ready.http.path: /ready
      com.openfaas.ready.http.initialDelaySeconds: 2
      com.openfaas.ready.http.periodSeconds: 2
```

For a slow initialisation, raise `initialDelaySeconds` so the first probe fires after the work is expected to have finished, rather than probing on a tight loop while the model is still loading:

```yaml
    annotations:
      com.openfaas.ready.http.path: /ready
      com.openfaas.ready.http.initialDelaySeconds: 60   # wait 60s before the first probe
      com.openfaas.ready.http.periodSeconds: 5
```

The full set of probe annotations (`timeoutSeconds`, `successThreshold`, `failureThreshold`) is listed under [Workloads](/reference/workloads/#custom-http-health-checks).

## Deploy and verify

```bash
faas-cli up
```

Confirm the function reports `Ready` before treating it as available:

```bash
faas-cli describe ready-example
```

To see the check working, scale to zero and invoke it — the first request must succeed rather than return a 500:

```bash
curl https://$OPENFAAS_URL/function/ready-example
```

## See also

* [Workloads: custom HTTP health checks](/reference/workloads/)
* [Health and readiness for functions](https://www.openfaas.com/blog/health-and-readiness-for-functions/)
