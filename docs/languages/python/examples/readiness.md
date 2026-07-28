Some functions need to do work before they can serve traffic: load a machine-learning model, warm a cache, open a database connection pool, or download a dataset. Until that work finishes, the function is running but not yet able to answer requests.

A readiness check lets OpenFaaS hold traffic back until the function signals it is ready. This matters every time a new Pod joins the service — a first deployment, a scale-up to add replicas, a rolling update, a restart, or a scale-from-zero. Without it, the first request to a Pod that is up but still initialising fails with a 500 or a timeout.

The watchdog ships a default readiness check, but it only reports that the watchdog process has started — not that your handler, or the dependencies behind it, can actually serve a request. Closing that gap is the developer's job: expose your own readiness path and have OpenFaaS probe it.

Use-cases:

* Loading an ML model or embeddings at start-up
* Warming an in-memory cache
* Opening a database or SDK connection pool
* Downloading reference data before the first request

!!! info "OpenFaaS Pro"
    Wiring the Kubernetes readiness probe to a custom path with the `com.openfaas.ready.http.*` annotations is part of [OpenFaaS Pro](/openfaas-pro/introduction). The `/ready` endpoint itself is ordinary handler code and runs anywhere.

## Overview

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

## Readiness is not health

Readiness and health (liveness) are different signals, and the difference matters:

* A failing **readiness** check takes the Pod out of the load-balancer rotation but leaves it running. Returning `503` from `/ready` for a while is a normal, deliberate signal — during start-up, or later if a downstream dependency drops out — and the Pod is put back as soon as the check passes again.
* A failing **health** check restarts the container.

You rarely need a custom health check — the watchdog's built-in liveness is enough — so put the effort into readiness. And return not-ready from your readiness path, never from health, when the function is merely warming up or temporarily unable to serve. Otherwise a slow start becomes a restart loop.

Keep the check on its own path, and keep it cheap. Do not point the probe at `/`: a probe-shaped request can fail what your handler expects, and it would run your real work on every probe — every couple of seconds — for nothing. A good check is a light touch: the init flag above, a quick `db.ping()`, or confirming the model object is loaded.

## Scaffold the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-http
faas-cli new --lang python3-http ready-example \
  --prefix ttl.sh/openfaas-examples
```

The example uses the public [ttl.sh](https://ttl.sh) registry — replace the prefix with your own registry for production use.

Update `ready-example/handler.py` with the code from the overview above.

## Configure the readiness probe

Point the readiness probe at the `/ready` path by adding these annotations to the function in `ready-example.yml`:

```yaml
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

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter ready-example \
 --tag digest
```

The function reports `Not Ready` until `initialize()` finishes, then `Ready`. Confirm before treating it as available:

```bash
faas-cli describe ready-example
```

Because the readiness probe holds traffic back until then, the first request to any freshly started Pod — whether from a first deployment, a scale-up, a rolling update, a restart, or a scale-from-zero — succeeds rather than returning a 500:

```bash
curl https://$OPENFAAS_URL/function/ready-example
```

## The other half: draining on the way down

Readiness governs a Pod joining the load balancer on the way up. Coming back down — a scale-down, a rolling update replacing the Pod, or a scale-to-zero — is a separate concern: the Pod should finish any in-flight requests before it is removed, rather than dropping them.

That is handled by the watchdog's graceful shutdown, not by the readiness check. On OpenFaaS Pro, the `write_timeout` environment variable sets how long Kubernetes waits for in-flight work to drain before removing the Pod. See [Custom TerminationGracePeriod](/reference/workloads/#custom-terminationgraceperiod-for-long-running-functions).

## See also

* [Workloads: custom HTTP health checks](/reference/workloads/)
* [Health and readiness for functions](https://www.openfaas.com/blog/health-and-readiness-for-functions/)
