# Scale to Zero

OpenFaaS Pro offers an additional component that can be used to scale idle functions to zero replicas. When scaled to zero, functions do not consume CPU or memory, and are then scaled back to the minimum desired amount of replicas upon first use.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

## Installation

Scale to Zero is enabled automatically when you install OpenFaaS Pro with helm, see [the helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

!!! info "Remember faas-idler?"

    If you worked with an older version of OpenFaaS Pro, then you'll remember faas-idler. It has now been replaced by the [OpenFaaS Pro autoscaler](/architecture/autoscaling) which performs both Horizontal Scaling of functions between their minimum and maximum replica count, and down to zero.

## Usage

Once enabled in your cluster, you can opt functions into being scaled to zero by adding the following label to stack.yml.

Create a new function:

```bash
export OPENFAAS_PREFIX=ghcr.io/openfaas
faas-cli new --lang go daily-job
```

Now add the labels:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  daily-job:
    labels:
      com.openfaas.scale.zero: true
    lang: go
    handler: ./daily-job
    image: ghcr.io/openfaas:daily-job
```

Once you have deployed and invoked your function, you should see it start to appear in the logs:

```bash
kubectl logs -n openfaas deploy/autoscaler -f
```

Open another terminal to monitor the replicas of your function:

```bash
kubectl get -n openfaas-fn deploy/daily-job -w
```

After the chosen scale to zero period, you will see its replicas scale to zero.

Then, you can invoke the function and you'll see the function scale up and get invoked again:

```bash
echo | faas-cli invoke daily-job
```

You can also deploy a store function, but bear in mind that they are not necessarily suitable for load testing:

```bash
faas-cli store deploy nodeinfo \
  --annotation com.openfaas.scale.zero=true
```

You can learn more about OpenFaaS auto-scaling here: [autoscaling](/architecture/autoscaling)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
