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

> Note: Setting `com.openfaas.scale.min` and `com.openfaas.scale.max` to the same value, allows to disable the auto-scaling functionality of openfaas.


For each alert fired the auto-scaler will add 5 replicas, but we are currently working on making the step configurable as a pre-defined step or proportional percentage. Once an alert is resolved due to a lower load than is needed for scaling the replica count will be scaled to the minimum replica count.

> Note: Active alerts can be viewed in the "Alerts" tab of Prometheus which is deployed with OpenFaaS.

## Scaling by CPU and/or memory utilization

When using Kubernetes the built-in [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) can be used instead of AlertManager.

Find out more in Stefan Prodan's blog post below:

https://stefanprodan.com/2018/kubernetes-scaleway-baremetal-arm-terraform-installer/#horizontal-pod-autoscaling

