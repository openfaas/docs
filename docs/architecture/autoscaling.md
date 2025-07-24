# Auto-scaling your functions

On this page:

* OpenFaaS Pro Autoscaler (below)
* [Legacy scaling for the Community Edition (CE)](#legacy-scaling-for-the-community-edition-ce)

## OpenFaaS Pro Autoscaler

!!! info "OpenFaaS Pro feature"
    This feature is included for [OpenFaaS Pro](/openfaas-pro/introduction) customers, and is designed for commercial use and production systems.

The [OpenFaaS Pro](/openfaas-pro/introduction/) Autoscaler scales functions horizontally between a minimum and maximum number of replicas, or to zero.

Watch Alex's overview of auto-scaling in OpenFaaS at KubeCon:

[![Live stream](https://img.youtube.com/vi/ka5QjX0JgFo/hqdefault.jpg)](https://www.youtube.com/watch?v=ka5QjX0JgFo)

Watch now: [How and Why We Rebuilt Auto-scaling in OpenFaaS with Prometheus](https://www.youtube.com/watch?v=ka5QjX0JgFo)

Configuration is via a label on the function.

| Label                                  | Description                                                                                                            | Default |
| :------------------------------------- | :--------------------------------------------------------------------------------------------------------------------- | :------ |
| `com.openfaas.scale.type`              | Scaling mode of `rps`, `capacity`, `cpu`, or the name of a custom rule added via the Helm chart.                       | `rps`   |
| `com.openfaas.scale.target`            | Target load per replica for scaling. This can also be set through the openfaas chart using `autoscaler.defaultTarget`. | `50`    |
| `com.openfaas.scale.target-proportion` | Proportion as a float of the target i.e. 0.9 = 90% of target.                                                          | `0.90`  |
| `com.openfaas.scale.min`               | The lower boundary for the number of Pods whilst under load.                                                           | `1`     |
| `com.openfaas.scale.max`               | The upper boundary for the number of Pods under maximum load. Or set to same as lower boundary to disable scaling.     | `20`    |
| `com.openfaas.scale.zero`              | Whether this function can be scaled to zero.                                                                           | `false` |
| `com.openfaas.scale.zero-duration`     | Idle duration before scaling to zero.                                                                                  | `15m`   |

All calls made through the gateway whether to a synchronous function `/function/` route or via the asynchronous `/async-function` route count towards this method of auto-scaling.

**Monitor what the autoscaler is doing**

The best way to monitor the load on functions, and the decisions of the autoscaler is through the Grafana dashboard which is included with OpenFaaS Pro. The overview dashboard shows everything you need to know at a glance across all functions, and the spotlight dashboard shows just the one function you select.

[![OpenFaaS Pro auto-scaling dashboard with Grafana](/images/grafana/overview-dashboard.png)](/openfaas-pro/grafana-dashboards)
> OpenFaaS Pro auto-scaling dashboard with Grafana

In addition to the dashboards, you can monitor the calculations and decisions of the autoscaler.

By default, only API calls to the scale endpoint will be logged, to reduce the load and the amount of logs that may need to be stored. Set `verbose` flag to `true` in the Helm chart under the `autoscaler` section to view each turn or iteration of the `[Scaler]` and `[Idler]` Goroutines as they make decisions.

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

* Static - disable scaling for a given function

  If the minimum and maximum scaling labels are set to the same value, then the function will be ignored by the autoscaler.

  This can be useful for advanced use-cases where a function may need to be stateful, act as a background worker, or where you are certain that scaling is not required.

* Capacity `capacity`

  Based upon inflight requests (or connections), ideal for: long-running functions or functions which can only handle a limited number of requests at once.

  A hard limit can be enforced through the `max_inflight` environment variable on the function, so the caller will need to retry the request some of the time. The OpenFaaS Pro queue-worker does this automatically, see also: [Retries](/openfaas-pro/retries).

* RPS `rps`

  Based upon requests per second completed by the function. A good fit for functions which execute quickly and have high throughput.

  You can tune this value on a per function basis.

* CPU `cpu`

  Based upon CPU usage of the function, this strategy is idea for CPU-bound workloads, or where Capacity and RPS are not giving the optimal scaling profile. The value configured here is in milli-CPU, so 1000 accounts for *1 CPU core*.

* Queue-depth `queue`

  Based upon the number of async invocations that are queued for a function. This allows you to scale functions rapidly and proactively to the desired number of replicas to process the queue as quickly as possible. Ideal for functions that are only invoked asynchronously. To use this mode, your [queue-worker](/openfaas-pro/jetstream) must be configured to scale consumers dynamically through the `function` mode.

* Custom metrics i.e. RAM, latency, application metrics, etc

  Functions can be scaled upon any custom metrics that are available in Prometheus, and which expose a "function_name label in the format of "name.namespace".

  This could include RAM usage, latency, business/application metrics, etc. Learn more: [Custom autoscaling rules](#custom-autoscaling-rules)

* Scaling to zero

  [Scaling to zero](/openfaas-pro/scale-to-zero) is an opt-in feature on a per function basis. It can be used in combination with any scaling mode, including *Static scaling*

## Examples of scaling modes

A quick primer on [hey](https://github.com/rakyll/hey) a load testing tool written in Go.

We maintain a forked version of hey with recent binaries at: [https://github.com/alexellis/hey/](https://github.com/alexellis/hey/).

* `-c` - concurrent connections
* `-z` - duration of the test as a Go duration
* `-t` timeout in seconds
* `q` - limit queries per second

You can install the forked version of `hey` via [arkade](https://arkade.dev) using: `arkade get hey`.

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

**4) Queue-depth based scaling**

Scaling based upon the queue depth for a function is a perfect match for asynchronous invocations.

Rather than measuring load upon the function as the other strategies do, the queue depth can be measured, and the number of target replicas can be set immediately.

This example limits concurrent requests to 1 for a long running sleep function.

```bash
faas-cli store deploy sleep \
--label com.openfaas.scale.max=10 \
--label com.openfaas.scale.target=1 \
--label com.openfaas.scale.type=queue \
--label com.openfaas.scale.target-proportion=1 \
--label com.openfaas.scale.zero=true \
--env max_inflight=1

hey -m POST -n 30 -c 30 \
  http://127.0.0.1:8080/async-function/sleep
```

The sleep function we've deployed has a hard limit that means it will only process 1 concurrent request at a time because of the `max_inflight` environment variable.

When 30 invocations are queued, the scaling parameters will mean that 30 replicas will be required to process the backlog, however the upper limit is 10 replicas. So it will scale to 10 replicas, and process up to queued requests in parallel.

The `com.openfaas.scale.zero=true` label is set to ensure that the function scales to zero when the queue is empty.

## Smoothing out scaling down with a stable window

If traffic to a function oscillates, the autoscaler will attempt to match that load and the number of replicas will also oscillate and mirror the load. This can be smoothed out through a stable window.

The `com.openfaas.scale.down.window` label can be set with a Go duration up to a maximum of `5m` or `300s`. When set, the autoscaler will record recommendations on each cycle, and only scale down a function to the highest recorded recommendation of replicas.

![Example of a stable window](/images/stable-window.png)
> There is variable load every 2.5 minutes, however the autoscaler does not scale down due to the stable window picking the highest recommendation over the past 5 minutes.

For example, a function receives a peak in traffic and scales to 10 replicas. The recommendations built up may include 8, 8, 7, 6, 5, 4, 5, 5 replicas, in this case, even if the autoscaler would pick something as low as 2 replicas, based upon the current load, it will only be allowed to scale down to 8 replicas. Once scale down window moves along, and the maximum recommendation decreases, then the value will eventually land on something that matches the current load being received.

In the above scenario, if you were to turn on verbose autoscaling, you'd have seen the following log message, showing the traffic demands 2x replicas, however the stable window is smoothing the decrease out.

```
2024/08/05 15:16:25 [Scaler] cows.openfaas-fn 10 => 2 (want: 8)
```

The purpose of this option is to slow down the rate of scaling down, when a function receives variable traffic over a relatively long period of time.

Scaling up, and scale to zero are unaffected, by default this setting is turned off.

## Custom autoscaling rules

In addition to the built-in scaling types, custom Prometheus expressions can be used to scale functions. For instance you may want to scale based upon queue-depth, Kafka consumer lag, latency, RAM used by a function (example in linked blog post), or a custom business metric exposed by your function's handler.

Blog post / walk-through: [How to scale OpenFaaS Functions with Custom Metrics](https://www.openfaas.com/blog/custom-metrics-scaling/).

For example, to add latency-based scaling using the gateway's gateway_functions_seconds histogram, you could add the following to the openfaas chart in values-pro.yaml:

```yaml
prometheus:
  recordingRules:
    - record: job:function_current_load:sum
      expr: |
        sum by (function_name) (rate(gateway_functions_seconds_sum{}[30s])) / sum by (function_name)  (rate( gateway_functions_seconds_count{}[30s]))
        and on (function_name) avg by(function_name) (gateway_service_target_load{scaling_type="latency"}) > bool 1
      labels:
        scaling_type: latency
```

To check the configuration of current recording rules use the Prometheus UI or run `kubectl edit -n openfaas configmap/prometheus-config` followed by `kubectl rollout restart -n openfaas deploy/prometheus`.

## Scaling to Zero

Scaling functions to zero replicas can improve efficiency and reduce costs in your Kubernetes clusters:

1. **Cost Savings**: By scaling down to zero when idle, you can reduce the number of nodes required in your cluster, leading to lower infrastructure costs with fewer, or smaller nodes required.
2. **Resource Efficiency**: Scaling down to zero helps to free up resources in your cluster, this also helps with on-premises clusters where the amount of nodes may be fixed.
3. **Security**: By scaling functions down, the attack surface is also reduced to only active functions.

### Scaling down to zero replicas

By default, no function will scale to zero unless you add the label `com.openfaas.scale.zero=true` to the function. This is a per-function setting, so you can choose which functions should scale down to zero.

Learn more: [Scale to Zero](/openfaas-pro/scale-to-zero)

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
    The Community Edition (CE) is meant for development only, or internal use in non-business use-cases.

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

> Note:
> Setting `com.openfaas.scale.min` and `com.openfaas.scale.max` to the same value, allows to disable the auto-scaling functionality of openfaas.
> Setting `com.openfaas.scale.factor=0` also allows to disable the auto-scaling functionality of openfaas.

For each alert fired the auto-scaler will add a number of replicas, which is a defined percentage of the max replicas. This percentage can be set using `com.openfaas.scale.factor`. For example setting `com.openfaas.scale.factor=100` will instantly scale to max replicas. This label enables to define the overall scaling behavior of the function.

> Note: Active alerts can be viewed in the "Alerts" tab of Prometheus which is deployed with OpenFaaS.
