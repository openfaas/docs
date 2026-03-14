# Extended timeouts

In this tutorial you'll learn how to extend the default timeout so that your functions can run for longer.

One of the most common support questions we get is about timeouts. Users often ask why their function is timing out after 30s or 60s.

!!! note "It's not us, it's you."

    We know it's frustrating, but we would know if there was a regression in OpenFaaS that meant timeouts stopped working as expected.

> For OpenFaaS Edge users should consult the eBook [Serverless For Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else) and update `docker-compose.yaml`.

## Why do functions timeout?

OpenFaaS functions can run for as long as necessary, some users have reported running executions for 48 hours or longer. The longest executions should be [run asynchronously](/reference/async/), so that the HTTP caller is not blocked waiting for a result.

Functions timeout due to one of the following:

* Using the default timeout set in the Helm chart in values.yaml for the gateway
* Using the default timeout in the Function's stack.yaml, or not setting all of the timeout environment variables
* An error in the function's code - a blocking I/O operation, a deadlock, or a crash/premature exit of the process
* Using an Ingress Controller or Load Balancer which has a low default timeout

Once you've followed all the instructions in this guide, make sure you've ruled out your Ingress Controller or Load Balancer before reaching out for help. For instance, [Ingress Nginx has a timeout set of 60 seconds](#load-balancers-ingress-and-service-meshes).

You can use the [following GitHub repository](https://github.com/alexellis/go-long) with three sample functions made with Go, Python and Node to confirm the issue isn't in your own function or code.

The [openfaas helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas) contains a detailed explanation of values for each OpenFaaS component including timeouts.

The most recent versions of OpenFaasS Standard and OpenFaaS For Enterprises, only need timeout values to be applied to the Gateway itself, they are then applied to the operator and built-in queue-worker by default.

For dedicated [queue-worker](https://github.com/openfaas/faas-netes/tree/master/chart/queue-worker) installations, you will need to configure their `upstreamTimeout` value separately in values.yaml


## Timeout reference

The function's `exec_timeout` must always be less than or equal to the gateway's `upstream_timeout`. If the function's timeout is larger than the gateway's, the gateway will cancel the request before the function finishes, resulting in a 502 or timeout error.

| Component | Variable | Set via |
|-----------|----------|---------|
| Gateway | `upstream_timeout` | Helm `gateway.upstreamTimeout` |
| Gateway | `read_timeout` | Helm `gateway.readTimeout` |
| Gateway | `write_timeout` | Helm `gateway.writeTimeout` |
| Function | `exec_timeout` | `stack.yaml` environment |
| Function | `read_timeout` | `stack.yaml` environment |
| Function | `write_timeout` | `stack.yaml` environment |

The function's `exec_timeout` must be equal to or shorter than the gateway's `upstream_timeout`. The gateway's `read_timeout` and `write_timeout` must be slightly longer than `upstream_timeout` to give the gateway time to handle the request cleanly.

Here is a valid configuration for a function that can run for up to 30 minutes - either synchronously or asynchronously.

Gateway (Helm values.yaml):

```yaml
gateway:
  upstreamTimeout: 30m
  readTimeout: 30m1s
  writeTimeout: 30m1s
```

Function (stack.yaml):

```yaml
functions:
  my-function:
    environment:
      exec_timeout: 30m
      read_timeout: 30m1s
      write_timeout: 30m1s
```

Note: setting `exec_timeout: 10m` on a function when the gateway's `upstream_timeout` is only `5m` will **not** give the function 10 minutes. The gateway will timeout and return an error after 5 minutes.

## Configure your function's timeout

OpenFaaS functions usually embed a component called the watchdog, which is responsible for implementing timeouts in a consistent way across different languages.

Partial example showing the timeouts:

```yaml
functions:
  job:
    environment:
      write_timeout: 5m1s
      read_timeout: 5m1s
      exec_timeout: 5m
```

For an example of Node, Python and Golang, see the: [go-long samples](https://github.com/alexellis/go-long).

## Load Balancers, Ingress, and service meshes

If you're using a load-balancer, ingress controller or service mesh, then you may need to check the timeouts for those components too.

To rule-out errors introduced by intermediate components, you should port-forward the OpenFaaS gateway service and invoke the function via its `http://127.0.0.1:8080` URL.

AWS EKS is configured to use an [Elastic Load Balancer (ELB)](https://aws.amazon.com/blogs/aws/elb-idle-timeout-control/) as its default, which has an "idle timeout" of 60 seconds. You can override this up to 60 minutes. As an alternative, the [AWS Load Balancer Controller for Kubernetes](https://kubernetes-sigs.github.io/aws-load-balancer-controller) can be used to provision an Application Load Balancer (ALB) or Network Load Balancer (NLB) instead.

Google Cloud's various Load Balancer options have their [own configuration options too](https://cloud.google.com/load-balancing/docs/https).

For Traefik, see [Configuring Traefik timeouts](#configuring-traefik-timeouts) below.

Ingress Nginx is now a retired project and should not be used for new installations. If you are still using it, set the `nginx.ingress.kubernetes.io/proxy-read-timeout` annotation to extend the timeout. This annotation is specified in seconds - for example, to extend the timeout to 30 minutes, use `nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"`.

### Configuring Traefik timeouts

Traefik has two separate sets of timeouts to be aware of:

**Client-to-Traefik (EntryPoints)** - configured in the static configuration (CLI flags or Helm values). Controls how long Traefik waits for the client to send a request or receive a response. The key fields are `readTimeout` (default 60s), `writeTimeout` (default 0s) and `idleTimeout` (default 180s). See [EntryPoints - RespondingTimeouts](https://doc.traefik.io/traefik/routing/entrypoints/#respondingtimeouts).

**Traefik-to-App (ServersTransport)** - configured in the dynamic configuration using a [ServersTransport CRD](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/http/serverstransport/), and referenced via the `traefik.ingress.kubernetes.io/service.serverstransport` annotation on the Ingress. By default there is no timeout on how long Traefik waits for a backend to respond (`responseHeaderTimeout` is 0s). Consider setting `responseHeaderTimeout` to match the gateway's `upstreamTimeout` so that Traefik returns a 504 quickly when a function hangs, rather than waiting indefinitely.

Finally, if you need to invoke a function for longer than one of your infrastructure components allows, then you should use an [asynchronous invocation](/reference/async). Asynchronous function invocations bypass these components because they are eventually invoked from the queue-worker, not the Internet. The queue-worker for OpenFaaS Standard will also retry invocations if required.

## Further support

Check the [troubleshooting guide](https://docs.openfaas.com/deployment/troubleshooting/).
