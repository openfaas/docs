# Scale to Zero

OpenFaaS Pro offers an additional component that can be used to scale idle functions to zero replicas. When scaled to zero, functions do not consume CPU or memory, and are then scaled back to the minimum desired amount of replicas upon first use.

Scaling to zero can save money by make your resources go further:

* Fewer nodes are required in your cluster
* If you have a limited pool of nodes, you can make more efficient use of them

Scaling to zero can also increase security:

* If any of your functions have a vulnerability, the attack surface is reduced to only the time they are running
* Each time the function runs, it may be run on a different node with a freshly pulled image

Read more on the blog:

* [Fine-tuning the cold-start in OpenFaaS](https://www.openfaas.com/blog/fine-tuning-the-cold-start/)
* [Dude, where's my cold-start?](https://www.openfaas.com/blog/what-serverless-coldstart/)

## Installation

Scale to Zero is enabled automatically when you install OpenFaaS Pro with helm and set `autoscaler.enabled: true`. You can see a sample configuration for [OpenFaaS Pro](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values-pro.yaml) here.

Scale to zero is controlled by the [OpenFaaS Pro autoscaler](/architecture/autoscaling) which also performs Horizontal Scaling of functions between the default or configured minimum and maximum replica counts.

## Usage

### Check the autoscaler is deployed

First of all, if you have access to "kubectl", check that the autoscaler is deployed and running:

```bash
# Check the deployment is present
kubectl get deploy -n openfaas autoscaler -o wide

# Start watching its logs
kubectl logs -n openfaas deploy/autoscaler \
  --since 30s -f
```

### Configure a function to scale to zero

By default, functions do not scale to zero, even when the OpenFaaS Pro autoscaler is installed. This is by design, and means that you need to opt-in each of your functions to scale down.

This is achieved through adding a label to the stack.yml file:

* `com.openfaas.scale.zero` - `true` or `false` - default is false
* `com.openfaas.scale.zero-duration` - the time in a Go duration where there are no requests incoming before scaling a function down i.e. `20m`, `1h`

### Try out an example function

The ttl.sh registry is a free service that can be used to publish a container image without needing to log into a registry.

Create a new function using the `go` template:

```bash
export OPENFAAS_PREFIX=ttl.sh/daily-job:1h
faas-cli new --lang go daily-job
```

Now add the labels from above, we'll use a 15 minute timeout.

You can specify a Go duration such as `5m`, `10m30s` or `1h`. It's recommended that scale down time is set to at least 5-10 minutes to prevent any thrashing that may occur if the traffic to a function is sporadic.

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  daily-job:
    lang: go
    handler: ./daily-job
    image: ttl.sh/daily-job:1h
    labels:
      com.openfaas.scale.zero: true
      com.openfaas.scale.zero-duration: 15m
```

Then build and deploy the function to your OpenFaaS Pro cluster:

```bash
# Build, push and deploy the function:
faas-cli up -f daily-job.yml

# Invoke the function:
echo | faas-cli invoke daily-job
```

Once you have deployed and invoked your function, you should see it start to appear in the logs of the autoscaler:

```bash
kubectl logs -n openfaas deploy/autoscaler \
  --since 30s -f
```

Open another terminal to monitor the replicas of your function:

```bash
kubectl get -n openfaas-fn \
  deploy/daily-job -w
```

After the chosen scale to zero period, you will see its replicas scale to zero.

Then, you can invoke the function and you'll see the function scale up and get invoked again:

```bash
echo | faas-cli invoke daily-job
```

If you're interested in seeing the various Kubernetes events involved in scaling from zero, you can run:

```bash
kubectl get event -n openfaas-fn -w
```

You can also deploy a test function from the OpenFaaS Store, but bear in mind that they are not necessarily suitable for load testing:

```bash
faas-cli store deploy nodeinfo \
  --label com.openfaas.scale.zero=true \
  --label com.openfaas.scale.zero-duration=15m
```

To disable scale to zero for any function, set `com.openfaas.scale.zero` to `false`, or don't add the label at all.

You can learn more about OpenFaaS auto-scaling here: [autoscaling](/architecture/autoscaling)

The time taken to scale up a function and have it ready to serve traffic is called a "cold-start". To learn what a cold-start is, and why they are present in Kubernetes, and how to minimise them in OpenFaaS, read: [Dude where's my coldstart?](https://www.openfaas.com/blog/what-serverless-coldstart/)

Cold-starts can be tuned through the OpenFaaS Helm chart: [Tuning function cold-starts](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#tuning-function-cold-starts)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Talk to us](https://openfaas.com/pricing/)
