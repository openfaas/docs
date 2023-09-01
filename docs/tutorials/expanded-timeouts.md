# Expanded timeouts

In this tutorial you'll learn how to expand the default timeouts of OpenFaaS to run your functions for longer.

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

## Further support

Check the [troubleshooting guide](https://docs.openfaas.com/deployment/troubleshooting/).
