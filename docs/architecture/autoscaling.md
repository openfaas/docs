# Auto-scaling

Auto-scaling in OpenFaaS allows a function to scale up or down depending on demand represented by different metrics.

## Scaling by requests per second

OpenFaaS ships with a single auto-scaling rule defined in the mounted configuration file for AlertManager. AlertManager reads usage (requests per second) metrics from Prometheus in order to know when to fire an alert to the API Gateway.

The API Gateway handles AlertManager alerts through its `/system/alert` route.

The auto-scaling provided by this method can be disabled by either deleting the AlertManager deployment or by scaling the deployment to zero replicas.

The AlertManager rules ([alert.rules](https://github.com/openfaas/faas/blob/master/prometheus/alert.rules.yml)) for Swarm can be viewed here and altered as a configuration map.

All calls made through the gateway whether to a synchronous function `/function/` route or via the asynchronous `/async-function` route count towards this method of auto-scaling.

### Min/max replicas

The minimum (initial) and maximum replica count can be set at deployment time by adding a label to the function.

* `com.openfaas.scale.min` - by default this is set to `1`, which is also the lowest value and unrelated to scale-to-zero

* `com.openfaas.scale.max` - the current default value is `20` for 20 replicas

* `com.openfaas.scale.factor` by default this is set to `20%` and has to be a value between 0-100 (including borders)

* `com.openfaas.scale.zero` - set to `true` for scaling to zero, faas-idler must also be deployed which is part of OpenFaaS PRO

> Note: 
Setting `com.openfaas.scale.min` and `com.openfaas.scale.max` to the same value, allows to disable the auto-scaling functionality of openfaas. 
Setting `com.openfaas.scale.factor=0` also allows to disable the auto-scaling functionality of openfaas.

For each alert fired the auto-scaler will add a number of replicas, which is a defined percentage of the max replicas. This percentage can be set using `com.openfaas.scale.factor`. For example setting `com.openfaas.scale.factor=100` will instantly scale to max replicas. This label enables to define the overall scaling behavior of the function.

> Note: Active alerts can be viewed in the "Alerts" tab of Prometheus which is deployed with OpenFaaS.

## Scaling by CPU and/or memory utilization

When using Kubernetes the built-in [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) can be used instead of AlertManager.

* Try the 2019 tutorial: [Kubernetes HPAv2 with OpenFaaS](/tutorials/kubernetes-hpa/).

* Stefan Prodan also wrote a blog post about [HPA with OpenFaaS in 2018](https://stefanprodan.com/2018/kubernetes-scaleway-baremetal-arm-terraform-installer/#horizontal-pod-autoscaling)

> Note: In addition to the above, both of the OpenFaaS watchdogs automatically provide custom metrics that can be used for HPAv2 scaling rules.

## Zero-scale

Scaling from zero is turned on by default, for any function or endpoint, this setting can be toggled on or off. Scaling to zero to recover idle resources is available in OpenFaaS, but is not turned on by default. There are two parts that make up scaling to zero or (zero-scale) in the project.

For a technical overview see the blog post: [Scale to Zero and Back Again with OpenFaaS](https://www.openfaas.com/blog/zero-scale/).

### Scaling up from zero replicas

Scaling up from zero replicas or 0/0 can be toggled through the `scale_from_zero` environment variable for the OpenFaaS Gateway. This is turned on by default on Kubernetes and faasd.

The latency between accepting a request for an unavailable function and serving the request is sometimes called a "Cold Start".

* What if I don't want a "cold start"?

The cold start in OpenFaaS is strictly optional and it is recommended that for time-sensitive operations you avoid one. This can be achieved by not scaling critical functions down to zero replicas, or by invoking them through the asynchronous route which decouples the request time from the caller.

* What exactly happens in a "cold start"?

The "Cold Start" consists of the following: creating a request to schedule a container on a node, finding a suitable node, pulling the Docker image and running the initial checks once the container is up and running. This "running" or "ready" state also has to be synchronised between all nodes in the cluster. The total value can be reduced by pre-pulling images on each node and by setting the Kubernetes Liveness and Readiness Probes to run at a faster cadence.

Instructions for optimizing for a low cold-start are provided in [the helm chart for Kubernetes](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

When `scale_from_zero` is enabled a cache is maintained in memory indicating the readiness of each function. If when a request is received a function is not ready, then the HTTP connection is blocked, the function is scaled to min replicas, and as soon as a replica is available the request is proxied through as per normal. You will see this process taking place in the logs of the *gateway* component.

### Scaling down to zero

Scaling down to zero replicas is also called "idling".

There are two approaches available for idling functions:

#### 1) faas-idler

You can use the faas-idler which is available with [OpenFaaS PRO](https://openfaas.com/support). `faas-idler` allows some basic presets to be configured and then monitors the built-in Prometheus metrics on a regular basis to determine if a function should be scaled to zero. Only functions with a label of `com.openfaas.scale.zero=true` are scaled to zero, all others are ignored. Functions are scaled to zero through the OpenFaaS REST API.

If you wish to only observe which functions would have been scaled down - pass the "-read-only" flag, or set this via the helm chart.

#### 2) OpenFaaS REST API

If you want to use your own set of criteria for idling functions then you can make use of the OpenFaaS REST API to decide when to scale functions to zero. You can build and deploy your own custom controller for this task.
