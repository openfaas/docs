## Function invocations

This page aims to explain function invocations and lifecycle.

In an OpenFaaS installation, functions can be invoked through HTTP requests to the OpenFaaS gateway, specifying the path as part of the URL.

There are some differences between OpenFaaS with [Kubernetes](/docs/deployment/kubernetes.md) and [faasd](/docs/deployment/faasd.md), so this page focuses on users of Kubernetes.

![Conceptual diagram: a synchronous invocation](/docs/images/invoke.png)
> Conceptual diagram: a synchronous invocation.

Each function is deployed as a Kubernetes Deployment and Service, with a number of replicas. Just like any other Kubernetes workload, it can be scaled up and down and handle multiple concurrent requests.

See also: [OpenFaaS Stack](/docs/architecture/stack.md)

## How is a function bundled?

OpenFaaS functions are built as OCI-compatible container images, which usually contain the OpenFaaS watchdog as a middleware or proxy. Existing containers that conform to the OpenFaaS workload definition can also be deployed.

Whenever you run `faas-cli build` or `faas-cli publish` using one of the OpenFaaS templates from the store, you'll find a container image built into your local library. The docker or buildkit CLI is used for this process, and means that CI/CD for functions can be very similar to other containers that you may be building already.

Templates tend to abstract away the Dockerfile and entry-point HTTP server from you, so that you can focus on writing a HTTP or function handler, they are still there however and you can look into the "template" folder to find them after running `faas-cli template store pull`

```bash
$ faas-cli template store pull node14
$ ls template/node14
Dockerfile      function        index.js        package.json    template.yml

$ ls template/node14/
handler.js      package.json
```

As explained on the link below, the Classic Watchdog can be used to turn CLIs or bash into HTTP endpoints for use as functions.

See also: [Watchdog](/docs/architecture/watchdog.md)

If you have existing containers that server HTTP traffic, then the chances are that OpenFaaS may be able to deploy them for you:

See also: [Workload definition](/docs/reference/workloads.md)

## What gets created when a function is deployed?

When using Kubernetes, each function you deploy through the OpenFaaS API will create a separate Kubernetes Deployment object. Deployment objects have a "replicas" value which corresponds to the number of Pods created in the cluster, that can serve traffic for your function.

A Kubernetes Service object is also created and is used to access the function's HTTP endpoint on port 80, within the cluster.

To see the resources created, you can type in:

```bash
faas-cli store deploy cows
kubectl get -n openfaas-fn service/cows -o yaml
kubectl get -n openfaas-fn deploy/cows -o yaml
```

By default, all functions have a minimum of 1 replica set [through auto-scaling labels](/docs/architecture/autoscaling.md), this can prevent a so called "cold-start", where a deployment is set to 0 replicas, and a Pod needs to be created to serve an incoming request.

The [Scale to Zero functionality of OpenFaaS Pro](/docs/openfaas-pro/scale-to-zero.md) can be used to scale idle functions down to 0 replicas, to save on resources.

## How many times can a function be invoked?

By default there is no limit on the amount of concurrent requests that your function's containers can process at once. If you wish to limit concurrency, you can set up the `max_inflight: N` environment variable, when the limit is met, the caller will receive a 429 status code and can retry after some time.

The OpenFaaS Pro queue-worker has a built-in retrying mechanism, which the caller can use to ensure that requests are retried a number of times before being discarded.

## How do synchronous invocations work?

During development you may invoke the OpenFaaS gateway using a HTTP request to `http://127.0.0.1:8080/function/NAME`, where `NAME` is the name of the function. When you move to production, you may have another layer between your users and the gateway such as a reverse proxy or Kubernetes Ingress Controller.

The connection between the caller and the function remains connected until the invocation has completed, or times out.

See the below for TLS termination, custom domains and mapping various functions to traditional REST paths:

* [TLS with OpenFaaS](/docs/reference/ssl/kubernetes-with-cert-manager.md)

## How to asynchronous invocations work?

With an asynchronous invocation, the HTTP request is enqueued to NATS, followed by an "accepted" header and call-id being returned to the caller. Next, at some time in the future, a separate queue-worker component dequeues the message and invokes the function synchronously.

There is never any direct connection between the caller and the function, so the caller gets an immediate response, and can subscribe for a response via a webhook when the result of the invocation is available.

See also: [asynchronous invocations in OpenFaaS](/docs/reference/async.md)

## What about events?

A number of event triggers are supported in OpenFaaS CE and OpenFaaS Pro. With each of them, a long-running daemon subscribes to a topic or queue, then when it receives messages looks up the relevant functions and invokes them synchronously or asynchronously.

Popular event sources include: [Cron](/docs/reference/cron.md), [Apache Kafka](/docs/openfaas-pro/kafka-events.md) and [AWS SQS](/docs/openfaas-pro/sqs-events.md).

See also: [Triggers](/docs/reference/triggers.md)

## What about authentication?

The OpenFaaS REST API is used to manage functions, it has basic authentication enabled by default, and we provide instructions to enable encryption with TLS and a reverse proxy.

OpenFaaS Pro offers authentication using JWT tokens obtained through an Open ID Connect (OIDC) flow and an Identity Provider (IdP).

* [TLS with OpenFaaS](/docs/reference/ssl/kubernetes-with-cert-manager.md)
* [Authentication with OIDC](/docs/openfaas-pro/sso.md)

For functions, you should provide your own authentication mechanism, such as a shared token, OIDC, HMAC or basic authentication.

## FAQ

If you have other questions on invocations and lifecycle, please feel free to reach out to us with suggestions via email.

### How do I limit how many requests run in a Pod?

As explained above, the environment variable `max_inflight` can limit concurrent requests.

You could set this value to 1, if you wanted to ensure that only one request executes in a Pod at a time, however it's recommended that you combine this with the back-off architecture to ensure requests are retried whilst waiting for scaling or free capacity.

See also: [How to process your data the resilient way with back pressure](https://www.openfaas.com/blog/limits-and-backpressure/)

### How to I retry failed requests?

See also: [How to process your data the resilient way with back pressure](https://www.openfaas.com/blog/limits-and-backpressure/) and [Retrying requests](/docs/openfaas-pro/retries.md)

### Can I bring an existing HTTP service to OpenFaaS?

You may be able to bring it straight over to OpenFaaS, if you can configure it to bind to HTTP port 8080 and you also configure a HTTP health check path, so that Kubernetes knows when it's ready to receive traffic.

If you cannot change your code, you may want to add the ["of-watchdog"](/docs/architecture/watchdog.md) to the Dockerfile.

See also: [Workload definition](/docs/reference/workloads.md)

### How do I enable isolation for multi-tenant workloads?

On Google Cloud, you can create a node-pool with gVisor and enable it.

For self-hosted Kubernetes, you may want to explore Kata containers.

With either, you then need to configure the Pod's "runtimeClass" setting, which is done with an [OpenFaaS Profile](/docs/reference/profiles.md).
