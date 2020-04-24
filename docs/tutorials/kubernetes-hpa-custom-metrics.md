# Kubernetes HPA with Custom Metrics

This guide enables Kubernetes HPAv2 (Horizontal Pod Autoscaling) with Custom Metrics.

The core components required are:

* Prometheus (deployed with OpenFaaS) - for scraping (collecting), storing and enabling queries
* [Prometheus Metrics Adapter](https://github.com/directxman12/k8s-prometheus-adapter) - to expose Prometheus metrics to the Kubernetes API server
* Helm for installing the metrics adapter

## Install the pre-reqs

* [Download Helm3](https://helm.sh/docs/intro/install/)

If you're an [arkade](https://get-arkade.dev/) user, helm is available at `~/.arkade/bin/helm3/`, add it to your path with `export PATH=$PATH:~/.arkade/bin/helm3/`.

* Install OpenFaaS via Helm or via [arkade](https://get-arkade.dev/)

You will need to use the latest version of the chart and faas-netes.

```bash
arkade install openfaas \
  --set gateway.directFunctions=false
```

Additionally, set `gateway.directFunctions=false` so that the provider (faas-netes) performs 
manual load-balancing between Pod IPs instead of relying on Kubernetes, which could use 
KeepAlive on the connections with the load-testing tool and not spread the load evenly.

* Install the [Prometheus Metrics Adapter](https://github.com/directxman12/k8s-prometheus-adapter)

Create a `values.yaml` file with overrides for Prometheus when deployed via OpenFaaS:

```yaml
prometheus:
  url: http://prometheus.openfaas.svc
  port: 9090
rules:
  default: false
  custom:
    - seriesQuery: 'http_requests_total{kubernetes_namespace!="",kubernetes_pod_name!=""}'
      resources:
        overrides:
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)'
```

Notes: the URL `http://prometheus.openfaas.svc` points to the specific Prometheus instance that OpenFaaS uses. The custom rule
maps `http_requests_total` to the `http_requests_per_second` metric in Kubernetes measuring over a `1m` timeframe.

## Deploy a function

Deploy the nodeinfo sample, which is a Node.js HTTP server to print system information about the container it is running in.

Add two annotations to enable Prometheus scraping, and set a min and max scale to the same number to disable OpenFaaS autoscaling.

```bash
faas-cli store deploy nodeinfo \
 --annotation prometheus.io.scrape=true \
 --annotation prometheus.io.port=8081 \
 --annotation com.openfaaas.scale.min=1 \
 --annotation com.openfaaas.scale.max=1
```

The OpenFaaS watchdog and classic watchdog both expose standard HTTP metrics on port 8081. You can change the port 
to another one and expose your own metrics, if you wish. You can also alter the path via `prometheus.io.path`.

> Note: if you are using an older OpenFaaS (faas-netes) version `< 0.10.4`, then you will need to manually edit the function's 
deployment and update the `prometheus.io.scrape` annotation from `false` to `true`.

## Generate some load

Use the hey tool to generate some low-level metrics:

```bash
hey -q 1 -c 1 -z 15m http://localhost:8080/function/nodeinfo
```

This sends 1 request / second over 15 minutes.

## Check the data is populated in Prometheus

Port-forward the Prometheus UI:

```bash
kubectl port-forward svc/prometheus -n openfaas 9090:9090
```

Look at the `http_requests_total` metrics, you should see data for both namespaces:

* `http_requests_total{kubernetes_namespace="openfaas"}`
* `http_requests_total{kubernetes_namespace="openfaas-fn"}`

## Create a HPAv2 rule

Create a rule to scale the function independently of the gateway's auto-scaling algorithm.

We need to reference the deployment created by OpenFaaS in the openfaas-fn namespace in 
the `scaleTargetRef` field so that the autoscaler knows which Deployment to scale.

Then specify the metricName and an `targetAverageValue`. If a Pod starts to process more
than `5` total requests per second, then the deployment will be scaled, until the average value 
is less than 5, or the upper ceiling of 10 pods is hit.

```yaml
cat >nodeinfo-hpa.yaml<<EOF
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nodeinfo-hpa
  namespace: openfaas-fn
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodeinfo
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metricName: http_requests_per_second
        targetAverageValue: 5
EOF
kubectl apply -f nodeinfo-hpa.yaml
```

Now ramp up the load-test using hey, so that the traffic is over 5 requests/per second.

```bash
hey -q 10 -c 1 -z 15m http://localhost:8080/function/nodeinfo
```

We should see at least 2-3 new pods come online to deal with the additional load, until 
the average between all is around 5 or lower.

## Monitor the auto-scaling

Run the following to watch the metrics being observed and new replicas of the 
nodeinfo deployment being brought online.

```bash
watch "kubectl describe -f nodeinfo-hpa.yaml"
```

You'll see output like below which gives reasoning on why changes are being made.

```bash
Name:                                  nodeinfo-hpa
Namespace:                             openfaas-fn
Labels:                                <none>
CreationTimestamp:                     Fri, 24 Apr 2020 13:11:30 +0100
Reference:                             Deployment/nodeinfo
Metrics:                               ( current / target )
  "http_requests_per_second" on pods:  1890m / 5
Min replicas:                          1
Max replicas:                          10
Deployment pods:                       5 current / 5 desired

Type     Reason                        Age                    From                       Message
----     ------                        ----                   ----                       -------
  Normal   SuccessfulRescale             61s                    horizontal-pod-autoscaler  New size: 5; reason: All metrics below target
```

You can also look at the [Prometheus query](http://localhost:9090/graph?g0.range_input=1h&g0.expr=http_requests_total%7Bkubernetes_namespace%3D%22openfaas-fn%22%2C%20faas_function%3D%22nodeinfo%22%7D&g0.tab=1) of `http_requests_total{kubernetes_namespace="openfaas-fn", faas_function="nodeinfo"}`.

View its rate over a minute with: `rate(http_requests_total{kubernetes_namespace="openfaas-fn", faas_function="nodeinfo"}[1m])`
