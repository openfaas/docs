# Expanded timeouts

In this tutorial you'll learn how to expand the default timeouts of OpenFaaS to run your functions for longer.

!!! note "It's not us, it's you."

    We would know if there was a regression in OpenFaaS that meant timeouts stopped working as expected.

    Typically, every time users reach out to us for help with timeouts it's due to a misconfiguration between one of the components in their stack, or a problem with the function itself.

    You will need to use our sample functions, which we know work, before reaching out for support.

## Part 1 - the core components

When running OpenFaaS on Kubernetes, you need to set various timeout values for the distributed components of the OpenFaaS control plane. These options are explained in the [helm chart README](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas). The easiest option for new users is to set them all to the same value.

 > For faasd users, you'll need to edit the equivalent fields in your `docker-compose.yaml` file, see the eBook [Serverless For Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else).

We will set:

* `gateway.upstreamTimeout`
* `gateway.writeTimeout`
* `gateway.readTimeout`

All timeouts are to be specified in Golang duration format i.e. `1m` or `60s`, or `1m30s`.

The `HARD_TIMEOUT` is set 1-2s higher than the `TIMEOUT` value since one needs to happen before the other.

```bash
export TIMEOUT=5m
export HARD_TIMEOUT=5m2s

arkade install openfaas \
  --set gateway.upstreamTimeout=$TIMEOUT \
  --set gateway.writeTimeout=$HARD_TIMEOUT \
  --set gateway.readTimeout=$HARD_TIMEOUT
```

Once installed with these settings, you can invoke functions for up to `5m` synchronously and asynchronously.

If using Helm or Argo CD, then add the following to your values.yaml file instead:

```yaml
gateway:
  writeTimeout: 5m1s
  readTimeout: 5m1s
  upstreamTimeout: 5m
```

## Part 2 - Your function's timeout

Now that OpenFaaS will allow a longer timeout, configure your function.

For the newer templates based upon HTTP which use the of-watchdog, adapt the following sample: [go-long: Golang function that runs for a long time](https://github.com/alexellis/go-long)

For classic templates using the classic watchdog, you can follow the workshop: [Lab 8 - Advanced feature - Timeouts](https://github.com/openfaas/workshop/blob/master/lab8.md)

If you're unsure which template you're using, check the source code of the Dockerfile in the `templates` folder when you build your functions.

## Load Balancers, Ingress, and service meshes

If you're using a load-balancer, ingress controller or service mesh, then you may need to check the timeouts for those components too.

To rule-out errors introduced by intermediate components, you should port-forward the OpenFaaS gateway service and invoke the function via its `http://127.0.0.1:8080` URL.

AWS EKS is configured to use an [Elastic Load Balancer (ELB)](https://aws.amazon.com/blogs/aws/elb-idle-timeout-control/) as its default, which has an "idle timeout" of 60 seconds. You can override this up to 60 minutes or switch to a different type of AWS load-balancer.

Google Cloud's various Load Balancer options have their [own configuration options too](https://cloud.google.com/load-balancing/docs/https).

Finally, if you need to invoke a function for longer than one of your infrastructure components allows, then you should use an [asynchronous invocation](/reference/async)

## Further support

Check the [troubleshooting guide](https://docs.openfaas.com/deployment/troubleshooting/).
