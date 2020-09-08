# Kubernetes HPAv2 with OpenFaaS

Kubernetes has a Pod auto-scaling mechanism built-in called Horizontal Pod Autoscaler (HPA) which can be used as an alternative to the built-in OpenFaaS auto-scaling.

## 1. Pre-reqs

### `hey` load-testing tool

We'll use the [hey](https://github.com/rakyll/hey) tool which also features in the OpenFaaS workshop for auto-scaling.

Download the latest release for your OS from this page and rename it to `hey` or `hey.exe`: [https://github.com/rakyll/hey/releases](https://github.com/rakyll/hey/releases)

### Helm

Some of the dependencies in this tutorial may rely on `helm`. The client-side CLI called `helm` is not insecure, but some people have concerns about using the server-side component named `tiller`. If you think that using helm's tiller component is insecure, then you should use the `helm template` command instead.

### Install OpenFaaS

Use the Deployment guide or an existing installation, you should also install the CLI.

### Install the metrics server

HPAv2 relies on the [Kubernetes metrics-server](https://github.com/kubernetes-incubator/metrics-server) which can be installed using a helm chart.

Most cloud providers are compatible with the metrics-server, but some are not, or require an additional "insecure" flag to be configured.

```sh
helm install --name metrics-server --namespace kube-system \
  stable/metrics-server
```

You can see the [Helm chart](https://github.com/helm/charts/tree/master/stable/metrics-server) for more options.

## 2. Check your metrics-server

When you see metrics appear from this command, continue with the tutorial.

```sh
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"

kubectl get --raw /metrics
```

Check the logs of the metrics-server for any errors:

```
kubectl logs deploy/metrics-server -n kube-system
```

If you don't see any metrics, then you may be using a cloud which needs the "insecure" work-around.

```sh
helm del --purge metrics-server

helm install --name metrics-server --namespace kube-system \
  stable/metrics-server \
  --set args="{--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname}"
```

You will now get Pod metrics along with Node metrics, these take a while to propagate.

Find out the CPU and memory usage across nodes:

```sh
kubectl top node
NAME   CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
nuc7   377m         4%     7955Mi          24%
```

Find out the usage across Pods:

```sh
# View the functions
kubectl top pod -n openfaas-fn
NAME                        CPU(cores)   MEMORY(bytes)   
nodeinfo-6f48f9b548-gbtr4   2m           3Mi

# View the core services
kubectl top pod -n openfaas
NAME                                 CPU(cores)   MEMORY(bytes)   
alertmanager-666c65c694-k8q6h        2m           10Mi            
basic-auth-plugin-6d97c6dc5b-rbw29   1m           4Mi             
faas-idler-67f9dcd4fc-85rbt          1m           4Mi             
gateway-7c687d498f-nvjh8             4m           21Mi            
nats-d4c9d8d95-fxrc2                 1m           6Mi             
prometheus-549c7d687d-zznmq          9m           41Mi            
queue-worker-544bcb7c67-72l72        1m           3Mi
```

> Note: you can view function usage using `-n openfaas-fn`, or core-service usage with `-n openfaas`

### Disable auto-scaling with OpenFaaS

You can either disable the OpenFaaS auto-scaling functionality completely, or just on a per-function basis

#### Disable auto-scaling from OpenFaaS on a per function basis

If you want to mix OpenFaaS auto-scaling and HPAv2, then add the additional label when deploying your functions:

```
faas-cli deploy --label com.openfaas.scale.factor=0
```

Or add the label to your `stack.yml` YAML file:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:31112
functions:
  nodeinfo:
    image: functions/nodeinfo:latest
    skip_build: true
    requests:
      cpu: 10m
    labels:
      com.openfaas.scale.factor: 0
```

#### Disable auto-scaling from OpenFaaS on all functions

Disable auto-scaling by scaling alertmanager down to zero replicas, this will stop it from firing alerts.

```sh
kubectl scale -n openfaas deploy/alertmanager --replicas=0
```

### Deploy your first function

For HPAv2 to work, we need to deploy a function with a minimum request value for CPU.

Let's create a `stack.yml` file:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:31112
functions:
  nodeinfo:
    image: functions/nodeinfo:latest
    skip_build: true
    requests:
      cpu: 10m
```

```sh
faas-cli deploy
```

Check that the CPU request was created on the Deployment:

```sh
kubectl describe deploy/nodeinfo -n openfaas-fn | grep cpu

cpu:        10m
```

Note: you can find the Docker image for other store functions use the following:

```sh
faas-cli store list

faas-cli store inspect FUNCTION_NAME
```

### Create a HPAv2 rule for CPU/memory

You can create a rule via YAML, or create one via the CLI:

```sh
kubectl autoscale deployment -n openfaas-fn \
  nodeinfo \
  --cpu-percent=50 \
  --min=1 \
  --max=10

horizontalpodautoscaler.autoscaling/nodeinfo autoscaled
```

* `-n openfaas-fn` refers to where the function is deployed
* `nodeinfo` is the name of the function
* `--cpu-percentage` is the level of CPU the Pod should reach before additional replicas are added
* `--min` minimum number of Pods
* `--max` maximum number of Pods

View the HPA record with:

```sh
kubectl get hpa/nodeinfo -n openfaas-fn

NAME       REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nodeinfo   Deployment/nodeinfo   <unknown>/50%   1         10        0          15s

# view the YAML

kubectl get hpa/nodeinfo -n openfaas-fn -o yaml
```

You can use `kubectl describe hpa/nodeinfo -n openfaas-fn` to get detailed information including any events such as scaling up and down.

```sh
kubectl describe hpa/nodeinfo -n openfaas-fn

Name:                                                  nodeinfo
Namespace:                                             openfaas-fn
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Sat, 10 Aug 2019 11:48:42 +0000
Reference:                                             Deployment/nodeinfo
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  20% (2m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       4 current / 4 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange   the desired count is within the acceptable range
Events:           <none>
```

### Generate some traffic

Use the following command with hey

```sh
export OPENFAAS_URL=http://127.0.0.1:31112

hey -c 5 \
 -z 5m \
 $OPENFAAS_URL/function/nodeinfo
```

* `-c` will simulate `5` concurrent users
* `-z` will run for `5m` and then complete

You should note that HPA is designed to react slowly to changes in traffic, both for scaling up and for scaling down. In some instances you may wait 20 minutes for all your Pods to scale back down to a normal level after the load has stopped.

Now in a new window monitor the progress:

```sh
kubectl describe hpa/nodeinfo -n openfaas-fn
```

Or in an automated fashion:

```sh
# watch -n 5 "kubectl describe hpa/nodeinfo -n openfaas-fn"
```

You can also monitor the replicas of your function in the OpenFaaS UI or via the CLI:

```sh
watch -n 5 "faas-cli list"
```

Here is an example of the replicas scaling up in response to the traffic created by `hey`:

```sh
Name:                                                  nodeinfo
Namespace:                                             openfaas-fn
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Sat, 10 Aug 2019 11:48:42 +0000
Reference:                                             Deployment/nodeinfo
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  12325% (1232m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       8 current / 10 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    SucceededRescale  the HPA controller was able to update the target scale to 10
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooManyReplicas   the desired replica count is more than the maximum replica count
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  18s   horizontal-pod-autoscaler  New size: 8; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  3s    horizontal-pod-autoscaler  New size: 10; reason: cpu resource utilization (percentage of request) above target
```

Note that whilst the scaling up was relatively quick, the scale-down may take significantly longer.

## 3. Wrapping up

In this tutorial we disabled the auto-scaling built into OpenFaaS which uses Prometheus and Alertmanager, and added in Kubernetes' own HPAv2 mechanism and its metrics-server.

### Notes and caveats

* Additional resource usage by Kubernetes

The additional services and workloads mentioned above do not come for free, and you may notice an increase in CPU and memory consumption across your cluster. Generally the auto-scaler in OpenFaaS is more efficient and light-weight.

* Scale to zero

You may not be able to scale your functions to zero if they are being managed by a HPAv2 rule with a minimum value of `1`, this is because after they are scaled down, the HPA rule will override the setting.

Nothing should block scaling up from zero, but HPAv2 is unlikely to allow any function to scale down and stay at `0/0` replicas

### Take it further

* Scale based upon RAM usage, instead of CPU
* Mix OpenFaaS auto-scaling with HPA for different use-cases
* Look into scaling with HPAv2 using "custom metrics" - [k8s-prom-hpa tutorial by Stefan Prodan](https://github.com/stefanprodan/k8s-prom-hpa)
