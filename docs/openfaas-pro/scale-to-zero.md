# Scale to Zero

OpenFaaS Pro offers an additional component that can be used to scale idle functions to zero replicas. When scaled to zero, functions do not consume CPU or memory, and are then scaled back to the minimum desired amount of replicas upon first use.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

## Installation

Scale to Zero is enabled automatically when you install OpenFaaS Pro with helm and set `autoscaler.enabled: true`. You can see a sample configuration for [OpenFaaS Pro](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values-pro.yaml) here.

!!! info "Remember faas-idler?"

    If you worked with an older version of OpenFaaS Pro, then you'll remember faas-idler. It has now been replaced by the [OpenFaaS Pro autoscaler](/architecture/autoscaling) which performs both Horizontal Scaling of functions between their minimum and maximum replica count, and down to zero.

## Usage

Once enabled in your cluster, you can opt functions into being scaled to zero by adding the following labels to stack.yml:

* `com.openfaas.scale.zero` - `true` or `false` - default is false
* `com.openfaas.scale.zero-duration` - the time in a Go duration where there are no requests incoming before scaling a function down i.e. `20m`, `1h`

Create a new function:

```bash
export OPENFAAS_PREFIX=ttl.sh/daily-job:1h
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
      com.openfaas.scale.zero-duration: 15m
    lang: go
    handler: ./daily-job
    image: ttl.sh/daily-job:1h
```

Then build and deploy the function:

```bash
faas-cli up -f daily-job.yml
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
  --label com.openfaas.scale.zero=true \
  --label com.openfaas.scale.zero-duration=15m
```

To disable scale to zero for any function, set `com.openfaas.scale.zero` to `false`, or don't add the label at all.

You can learn more about OpenFaaS auto-scaling here: [autoscaling](/architecture/autoscaling)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
