# Expanded timeouts

In this tutorial you'll learn how to expand the default timeouts of OpenFaaS to run your functions for longer.

## Part 1 - the core components

When installing with Kubernetes, you need to set various timeout values for the distributed components of the OpenFaaS control plane.

These options are explained in the [helm chart README](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas). The easiest option for new users is to set them all to the same value.

We will set:

* `gateway.upstreamTimeout`
* `gateway.writeTimeout`
* `gateway.readTimeout`
* `faasnetes.writeTimeout`
* `faasnetes.readTimeout`

For async tasks, also set:

* `queueWorker.ackWait`

All timeouts are to be specified in Golang duration format i.e. `1m` or `12s`, or `1m12s`.

```bash
export TIMEOUT=2m

arkade install openfaas \
  --set gateway.upstreamTimeout=$TIMEOUT \
  --set gateway.writeTimeout=$TIMEOUT \
  --set gateway.readTimeout=$TIMEOUT \
  --set faasnetes.writeTimeout=$TIMEOUT \
  --set faasnetes.readTimeout=$TIMEOUT \
  --set queueWorker.ackWait=$TIMEOUT
```

One installed with these settings, you can invoke functions for up to `2m` synchronously and asynchronously.

## Part 2 - Your function's timeout

Now that OpenFaaS will allow a longer timeout, configure your function.

For classic templates using the classic watchdog, you can follow the workshop: [Lab 8 - Advanced feature - Timeouts](https://github.com/openfaas/workshop/blob/master/lab8.md)

For the newer templates based upon HTTP which use the of-watchdog, adapt the following sample: [go-long: Golang function that runs for a long time](https://github.com/alexellis/go-long)

## Further support

Check the [troubleshooting guide](https://docs.openfaas.com/deployment/troubleshooting/) and work through the exercises above.

For Docker Swarm, simply edit `docker-compose.yml` and redeploy the stack.
