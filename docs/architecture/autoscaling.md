# Auto-scaling your functions

On this page:

* OpenFaaS Pro Autoscaler (below)
* [Alert-manager based auto-scaling for Community Edition](#legacy-scaling-for-the-community-edition-ce)

## OpenFaaS Pro Autoscaler

!!! info "OpenFaaS Pro feature"
    This feature is included for [OpenFaaS Pro](/openfaas-pro/introduction) customers, and is designed for commercial use and production systems.

The [OpenFaaS Pro](/openfaas-pro/introduction/) Scaler scales functions horizontally between a minimum and maximum number of replicas, or to zero.

Watch Alex's overview of auto-scaling in OpenFaaS at KubeCon:

[![Live stream](https://img.youtube.com/vi/ka5QjX0JgFo/hqdefault.jpg)](https://www.youtube.com/watch?v=ka5QjX0JgFo)

Watch now: [How and Why We Rebuilt Auto-scaling in OpenFaaS with Prometheus](https://www.youtube.com/watch?v=ka5QjX0JgFo)

Configuration is via a label on the function.

| Label                                  | Description                                                   | Default |
| :------------------------------------- | :------------------------------------------------------------ | :------ |
| `com.openfaas.scale.max`               | The maximum number of replicas to scale to.                   | `20`    |
| `com.openfaas.scale.min`               | The minimum number of replicas to scale to.                   | `1`     |
| `com.openfaas.scale.zero`              | Whether to scale to zero.                                     | `false` |
| `com.openfaas.scale.zero-duration`     | Idle duration before scaling to zero                          | `15m`   |
| `com.openfaas.scale.target`            | Target load per replica for scaling                           | `50`    |
| `com.openfaas.scale.target-proportion` | Proportion as a float of the target i.e. 1.0 = 100% of target | `0.90`  |
| `com.openfaas.scale.type`              | Scaling mode of `rps`, `capacity`, `cpu`                      | `rps`   |

All calls made through the gateway whether to a synchronous function `/function/` route or via the asynchronous `/async-function` route count towards this method of auto-scaling.

![OpenFaaS Pro auto-scaling dashboard with Grafana](https://pbs.twimg.com/media/FJ9EBVdWQAM9DeW?format=jpg&name=medium)
> OpenFaaS Pro auto-scaling dashboard with Grafana

## How auto-scaling works

There are three auto-scaling modes described in the next section. This is how they work.

* When configuring auto-scaling for a function, you need to set a target number which is the average load per replica of your function.
* Each mode can be used to record a current load for a function across all replicas in the OpenFaaS cluster.

Then, a query is run periodically to calculate the current load.

The current load is used to calculate the new number of replicas.

```
desired = ready pods * ( mean load per pod / target load per pod )
```

The target-proportion flag can be used to adjust how early or late scaling occurs:


```
desired = ready pods * ( mean load per pod / ( target load per pod * target-proportion ) )
```

For example:

* `sleep` is running in the `capacity` mode and has a target load of 5 in-flight requests.
* The load on the sleep function is measured as `15` inflight requests.
* There is only one replica of the `sleep` function because its minimum range is set to `1`.
* We are assuming `com.openfaas.scale.target-proportion` is set to 1.0 (100%).

```
mean per pod = 15 / 1

3 = ceil ( 1 * ( 15 / 5 * 1 ) )
```

Therefore, 3 replicas will be set.

With 3 replicas and 25 ongoing requests, the load will be spread more evenly, and evaluate as follows:

```
mean per pod = 25 / 3 = 8.33

5 =  ceil( 3 * ( 8.33 / 5 * 1 ) )
```

When the load is no longer present, it will evaluate as follows:

```
mean per pod = 0 / 3 = 0

0 = ceil ( 3 * ( 0 / 5 * 1) )
```

But the function will not be set to zero yet, it will be brought up to the minimum range which is 1.

Scaling to zero is based upon traffic observed from the gateway within a set period of time defined via `com.openfaas.scale.zero-duration`.

If you are limiting how much concurrency goes to a function, let's say for 100 requests maximum, then you may want to set the target to 100 with a proportion of 0.7, in this instance, when there are 70 ongoing requests, the autoscaler will add more replicas:


```
total load = 90
mean per pod = 90 / 1 = 90

2 =  ceil( 1 * ( 90 / ( 100 * 0.7 ) ) )
```

## Scaling modes

* Capacity `capacity`

  Based upon inflight requests (or connections), ideal for: long-running functions or functions which can only handle a limited number of requests at once.
  
  A hard limit can be enforced through the `max_inflight` environment variable on the function, so the caller will need to retry the request some of the time. The OpenFaaS Pro queue-worker does this automatically, see also: [Retries](/openfaas-pro/retries).

* RPS `rps`

  Based upon requests per second completed by the function. A good fit for functions which execute quickly and have high throughput.

  You can tune this value on a per function basis.

* CPU `cpu`

  Based upon CPU usage of the function, this strategy is idea for CPU-bound workloads, or where Capacity and RPS are not giving the optimal scaling profile. The value configured here is in milli-CPU, so 1000 accounts for *1 CPU core*.

* Scaling to zero

  Scaling to zero is an opt-in feature on a per function basis. It can be used in combination with any of the three scaling modes listed above.

## Testing out the various modes

A quick primer on [hey](https://github.com/rakyll/hey) a load testing tool written in Go.

* `-c` - concurrent connections
* `-z` - duration of the test as a Go duration
* `-t` timeout in seconds
* `q` - limit queries per second

You can install hey with [arkade](https://arkade.dev) using: `arkade get hey`.

**1) Capacity-based scaling:**

This function takes 1-2 seconds to complete, and uses a target, or *soft limit* of 5 concurrent requests.

```bash
# target: 5 inflight
# 100% utilization of target

faas-cli store deploy sleep \
--label com.openfaas.scale.max=10 \
--label com.openfaas.scale.target=5 \
--label com.openfaas.scale.type=capacity \
--label com.openfaas.scale.target-proportion=1.0 \
--label com.openfaas.scale.zero=true \
--label com.openfaas.scale.zero-duration=5m

# With a timeout of 10 seconds
# Run for 3 minutes
# With 5 concurrent callers
# Limited to 5 QPS per caller
hey -t 10 -z 3m -c 5 -q 5 \
  http://127.0.0.1:8080/function/sleep
```

To apply a hard limit, add `--env max_inflight=5` to the `faas-cli store deploy` command. 

What if you need to limit a function to processing only one request at a time?

Change the target to `1`, `target-proportion` to `0.95` and set the `max_inflight` to `1`.

```bash
faas-cli store deploy sleep \
--label com.openfaas.scale.max=10 \
--label com.openfaas.scale.target=1 \
--label com.openfaas.scale.type=capacity \
--label com.openfaas.scale.target-proportion=0.95 \
--env max_inflight=1

# With a timeout of 10 seconds
# Run for 3 minutes
# With 5 concurrent callers
# Limited to 5 QPS per caller
hey -t 10 -z 3m -c 5 -q 5 \
  http://127.0.0.1:8080/function/sleep
```

You'll see the replicas scale up to 5 over time, and back to 1 when the test is complete. No pod will serve more than one request at a time.

**2) RPS-based scaling:**

```bash
# target: 50 RPS
# 90% utilization of target

faas-cli store deploy nodeinfo \
--label com.openfaas.scale.max=10 \
--label com.openfaas.scale.target=50 \
--label com.openfaas.scale.type=rps \
--label com.openfaas.scale.target-proportion=0.90 \
--label com.openfaas.scale.zero=true \
--label com.openfaas.scale.zero-duration=10m

# Run for 3 minutes
# With 5 concurrent callers
# Limited to 20 QPS per caller
hey -z 3m -c 5 -q 20 \
  http://127.0.0.1:8080/function/nodeinfo
```

**3) CPU-based scaling:**

```bash
# target: 100 Mi
# 50% utilization of target

faas-cli store deploy figlet \
--label com.openfaas.scale.max=10 \
--label com.openfaas.scale.target=100 \
--label com.openfaas.scale.type=cpu \
--label com.openfaas.scale.target-proportion=0.50 \
--label com.openfaas.scale.zero=true \
--label com.openfaas.scale.zero-duration=30m

# Run for 3 minutes
# With 5 concurrent callers
# Limited to 20 QPS per caller
hey -m POST -d data -z 3m -c 5 -q 20 \
  http://127.0.0.1:8080/function/figlet
```

**3) CPU-based scaling w/o scale to zero:**

```bash
# target: 50 Mi
# 90% utilization of target

faas-cli store deploy cows \
--label com.openfaas.scale.max=5 \
--label com.openfaas.scale.target=50 \
--label com.openfaas.scale.type=cpu \
--label com.openfaas.scale.target-proportion=0.70 \
--label com.openfaas.scale.zero=false

hey -m POST -d data -z 3m -c 5 -q 10 \
  http://127.0.0.1:8080/function/cows
```

Note that `com.openfaas.scale.zero=false` is a default, so this is not strictly required.

## Scaling to Zero aka "Zero-scale"

Scaling functions to zero replicas when idle can save on costs by reducing the amount of nodes required in your cluster. You can also reduce the consumption of nodes on statically-sized or on-premises clusters.

In OpenFaaS, scaling to zero is turned off by default, and is part of the OpenFaaS Pro bundle and configured in the helm chart. Once installed, idle functions can be configured to scale down when they haven't received any requests for a period of time. We suggest that you set this figure to 2x the maximum timeout, or use the default timeout value if that makes sense for most of your functions.

### Scaling up from zero replicas

Scaling up from zero replicas or 0/0 can be toggled through the `scale_from_zero` environment variable for the OpenFaaS Gateway. This is turned on by default on Kubernetes and faasd.

The latency between accepting a request for an unavailable function and serving the request is sometimes called a "Cold Start".

* What if I don't want a "cold start"?

    The cold start in OpenFaaS is strictly optional and it is recommended that for time-sensitive operations you avoid one by having a minimum scale of 1 or more replicas. This can be achieved by not scaling critical functions down to zero replicas, or by invoking them through the asynchronous route which decouples the request time from the caller.

* What exactly happens in a "cold start"?

    The "Cold Start" consists of the following: creating a request to schedule a container on a node, finding a suitable node, pulling the Docker image and running the initial checks once the container is up and running. This "running" or "ready" state also has to be synchronised between all nodes in the cluster. The total value can be reduced by pre-pulling images on each node and by setting the Kubernetes Liveness and Readiness Probes to run at a faster cadence.

    Instructions for optimizing for a low cold-start are provided in [the helm chart for Kubernetes](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

    When `scale_from_zero` is enabled a cache is maintained in memory indicating the readiness of each function. If when a request is received a function is not ready, then the HTTP connection is blocked, the function is scaled to min replicas, and as soon as a replica is available the request is proxied through as per normal. You will see this process taking place in the logs of the *gateway* component.

    For an overview of cold-starts in OpenFaaS see: [Dude where's my coldstart?](https://www.openfaas.com/blog/what-serverless-coldstart/)

* What if my function is still running when it gets scaled down?

    That shouldn't happen, providing that you've set an adequate value for the idle detection for your function. But if it does, the OpenFaaS watchdog and our official function templates will allow a graceful termination of the function. See also: [Improving long-running jobs for OpenFaaS users](https://www.openfaas.com/blog/long-running-jobs/)

## Legacy scaling for the Community Edition (CE)

!!! warning "Legacy scaling for the Community Edition (CE)"
    The Community Edition (CE) of OpenFaaS uses legacy scaling technology, which is meant for development only.

A single auto-scaling rule defined in the mounted configuration file for AlertManager, which is used for all functions. AlertManager reads usage (requests per second) metrics from Prometheus in order to know when to fire an alert to the API Gateway.

The API Gateway handles AlertManager alerts through its `/system/alert` route.

The auto-scaling provided by this method can be disabled by either deleting the AlertManager deployment or by scaling the deployment to zero replicas.

The [AlertManager rule](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/templates/prometheus-cfg.yaml#L164) used in the Community Edition can be viewed here.

All calls made through the gateway whether to a synchronous function `/function/` route or via the asynchronous `/async-function` route count towards this method of auto-scaling.

### Min/max replicas

The minimum (initial) and maximum replica count can be set at deployment time by adding a label to the function.

* `com.openfaas.scale.min` - by default this is set to `1`, which is also the lowest value and unrelated to scale-to-zero

* `com.openfaas.scale.max` - the default and maximum value is `5` for 5/5 Pods

* `com.openfaas.scale.factor` by default this is set to `20%` and has to be a value between 0-100 (including borders)

* `com.openfaas.scale.zero` - set to `true` for scaling to zero, faas-idler must also be deployed which is part of OpenFaaS Pro

> Note:
> Setting `com.openfaas.scale.min` and `com.openfaas.scale.max` to the same value, allows to disable the auto-scaling functionality of openfaas. 
> Setting `com.openfaas.scale.factor=0` also allows to disable the auto-scaling functionality of openfaas.

For each alert fired the auto-scaler will add a number of replicas, which is a defined percentage of the max replicas. This percentage can be set using `com.openfaas.scale.factor`. For example setting `com.openfaas.scale.factor=100` will instantly scale to max replicas. This label enables to define the overall scaling behavior of the function.

> Note: Active alerts can be viewed in the "Alerts" tab of Prometheus which is deployed with OpenFaaS.

