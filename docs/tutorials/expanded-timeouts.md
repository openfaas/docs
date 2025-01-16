# Extended timeouts

In this tutorial you'll learn how to extend the default timeout so that your functions can run for longer.

One of the most common support questions we get is about timeouts. Users often ask why their function is timing out after 30s or 60s.

!!! note "It's not us, it's you."

    We know it's frustrating, but we would know if there was a regression in OpenFaaS that meant timeouts stopped working as expected.

## Why do functions timeout?

OpenFaaS functions can run for as long as necessary, some users have reported running executions for 48 hours or longer. The longest executions should be [run asynchronously](), so that the HTTP caller is not blocked waiting for a result.

Functions timeout due to one of the following:

* Using the default timeout set in the Helm chart in values.yaml for the gateway
* Using the default timeout in the Function's stack.yaml, or not setting all of the timeout environment variables
* An error in the function's code - a blocking I/O operation, a deadlock, or a crash/premature exit of the process
* Using an Ingress Controller or Load Balancer which has a low default timeout

Once you've followed all the instructions in this guide, make sure you've ruled out your Ingress Controller or Load Balancer before reaching out for help. For instance, [Ingress Nginx has a timeout set of 60 seconds](#load-balancers-ingress-and-service-meshes).

You can use the [following GitHub repository](https://github.com/alexellis/go-long) with three sample functions made with Go, Python and Node to confirm the issue isn't in your own function or code.

## Part 1 - the core components

When running OpenFaaS on Kubernetes, it is possible to override the timeout on various components of the OpenFaaS gateway, however it is only usually necessary to set the timeout on the gateway itself.

You can find the various options in the [helm chart README](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

For faasd users, you'll need to edit the equivalent fields in your `docker-compose.yaml` file, see the eBook [Serverless For Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else).

We need to set the following in the Helm chart's values.yaml file for the OpenFaaS chart:

* `gateway.upstreamTimeout`
* `gateway.writeTimeout`
* `gateway.readTimeout`

All timeouts are to be specified in Golang duration format i.e. `1m` or `60s`, or `1m30s`.

If using Helm or Argo CD, then add the following to your values.yaml file:

```yaml
gateway:
  writeTimeout: 5m1s
  readTimeout: 5m1s
  upstreamTimeout: 5m
```

Note that `upstreamTimeout` must always be lower than `writeTimeout` and `readTimeout`, to allow the gateway to handle the request before the HTTP server cancels the request.

If using arkade, you can run the following:

```bash
export TIMEOUT=5m
export SERVER_TIMEOUT=5m2s

arkade install openfaas \
  --set gateway.upstreamTimeout=$TIMEOUT \
  --set gateway.writeTimeout=$SERVER_TIMEOUT \
  --set gateway.readTimeout=$SERVER_TIMEOUT
```

Once installed with these settings, you can invoke functions for up to `5m` synchronously and asynchronously.

## Part 2 - Your function's timeout

Now that OpenFaaS will allow a longer timeout, configure your function.

OpenFaaS functions usually embed a component called the watchdog, which is responsible for implementing timeouts in a consistent way across different languages. Most templates use the newer of-watchdog, but a few may still be using the classic watchdog for compatibility reasons.

For the newer templates based upon HTTP which use the of-watchdog, adapt the following sample: [go-long: Golang function that runs for a long time](https://github.com/alexellis/go-long)

For classic templates using the classic watchdog, you can follow the workshop: [Lab 8 - Advanced feature - Timeouts](https://github.com/openfaas/workshop/blob/master/lab8.md)

If you're unsure which template you're using, check the source code of the Dockerfile in the `templates` folder when you build your functions, you should see a `FROM` line at the top of the file that will specify one or the other.

## Load Balancers, Ingress, and service meshes

If you're using a load-balancer, ingress controller or service mesh, then you may need to check the timeouts for those components too.

To rule-out errors introduced by intermediate components, you should port-forward the OpenFaaS gateway service and invoke the function via its `http://127.0.0.1:8080` URL.

AWS EKS is configured to use an [Elastic Load Balancer (ELB)](https://aws.amazon.com/blogs/aws/elb-idle-timeout-control/) as its default, which has an "idle timeout" of 60 seconds. You can override this up to 60 minutes. As an alternative, the [AWS Load Balancer Controller for Kubernetes](https://kubernetes-sigs.github.io/aws-load-balancer-controller) can be used to provision an Application Load Balancer (ALB) or Network Load Balancer (NLB) instead.

Google Cloud's various Load Balancer options have their [own configuration options too](https://cloud.google.com/load-balancing/docs/https).

For Ingress Nginx, set the `nginx.ingress.kubernetes.io/proxy-read-timeout` annotation to extend the timeout. This annotation is specified in seconds - for example, to extend the timeout to 30 minutes, use `nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"`.

Finally, if you need to invoke a function for longer than one of your infrastructure components allows, then you should use an [asynchronous invocation](/reference/async). Asynchronous function invocations bypass these components because they are eventually invoked from the queue-worker, not the Internet. The queue-worker for OpenFaaS Standard will also retry invocations if required.

## Further support

Check the [troubleshooting guide](https://docs.openfaas.com/deployment/troubleshooting/).
