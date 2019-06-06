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

* `com.openfaas.scale.min` By default this is set to `1`

* `com.openfaas.scale.max` The current default value is `20` for 20 replicas

* `com.openfaas.scale.factor` By default this is set to `20%` and has to be a value between 0-100 (including borders)

> Note: 
Setting `com.openfaas.scale.min` and `com.openfaas.scale.max` to the same value, allows to disable the auto-scaling functionality of openfaas. 
Setting `com.openfaas.scale.factor=0` also allows to disable the auto-scaling functionality of openfaas.


For each alert fired the auto-scaler will add a number of replicas, which is a defined percentage of the max replicas. This percentage can be set using `com.openfaas.scale.factor`. For example setting `com.openfaas.scale.factor=100` will instantly scale to max replicas. This label enables to define the overall scaling behavior of the function.

> Note: Active alerts can be viewed in the "Alerts" tab of Prometheus which is deployed with OpenFaaS.

## Scaling by CPU and/or memory utilization

When using Kubernetes the built-in [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) can be used instead of AlertManager.

Find out more in Stefan Prodan's blog post below:

https://stefanprodan.com/2018/kubernetes-scaleway-baremetal-arm-terraform-installer/#horizontal-pod-autoscaling

In addition to the above, both of the OpenFaaS watchdogs automatically provide custom metrics that can be used for HPAv2 scaling rules.

## Zero-scale

Scaling to zero is available in OpenFaaS but is not turned on by default. There are two parts that make up scaling to zero or (zero-scale) in the project. You can read more about how this works on the [OpenFaaS blog](https://www.openfaas.com/blog/zero-scale/).

### Scaling up from zero replicas

Scaling up from zero replicas or 0/0 can be configured through an optional flag in the API Gateway.

To turn on scaling from 0 to your minimum replica count set `zero_scale` to true for the gateway deployment using Helm or the `docker-compose.yml`.

The latency between accepting a request for an unavailable function and serving the request is sometimes called a "Cold Start". The Cold Start varies depending on whether you are using Kubernetes or Swarm and whether the image is pre-pulled on any Nodes in the cluster. For Kubernetes (faas-netes) you can fine-tune the function's intial check delays to reduce the total latency.

When `zero_scale` is enabled then each HTTP connection is blocked, the function is scaled to min replicas, and as soon as a replica is available the request is proxied through as per normal.

### Scaling down to zero

Scaling down to zero replicas is also called "idling". There are two options available for idling functions.

* Option 1 - faas-idler

You can use the [faas-idler](https://github.com/openfaas-incubator/faas-idler) project which is incubating in the openfaas-incubator organisation. faas-idler allows some basic presents to be configured and then monitors the built-in Prometheus metrics on a regular basis and then uses the OpenFaaS REST API to scale idle functions to zero replicas.

The [faas-idler](https://github.com/openfaas-incubator/faas-idler) component is easy to deploy with a docker-compose.yml file or Kubernetes YAML file.

* Option 2 - OpenFaaS REST API

If you want to use your own set of criteria for idling functions then you can make use of the OpenFaaS REST API to decide when to scale functions to zero. You can build and deploy your own custom controller for this task.
