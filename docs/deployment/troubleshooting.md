# Troubleshooting

There are two target platforms for OpenFaaS, and depending on which you are using, some of the instructions will differ.

## faasd

The troubleshooting instructions for [faasd](https://github.com/openfaas/faasd) are only available in the [eBook and manual](https://gumroad.com/l/serverless-for-everyone-else).

## OpenFaaS on Kubernetes

Kubernetes is a complex distributed system, and there are many things that can cause friction for new OpenFaaS users. This guide is devoted to helping you help yourself.

If you want to ask for help, make sure that you have run all of the commands below before doing so.

### The Config Checker

We recommend that all users run our automated config-checker tool which will help you to identify common problems with timeouts, function configuration and the core components. The tool is designed for OpenFaaS Standard and OpenFaaS for Enterprises, but should still give some useful output for OpenFaaS CE users.

* [openfaas/config-checker](https://github.com/openfaas/config-checker)

If we've asked you to run the config-checker via email or Slack, then please also collect the logs and output from kubectl by running our [openfaas-diagnostics.sh](https://github.com/openfaas/config-checker/blob/master/openfaas-diagnostics.sh) bash script. Send over the resulting openfaas.tgz file to our team.

### OpenFaaS didn't start

Look for `0/1`, restarts or errors showing up here:

```bash
kubectl get deploy -n openfaas
```

Next, try to describe one of the deployments such as the `gateway`:

```bash
kubectl describe -n openfaas deploy/gateway
```

Next, check the logs for the specific deployment that is failing to start:

```bash
kubectl logs -n openfaas deploy/gateway
```

Next, check the events in the openfaas namespace:

```bash
kubectl get events -n openfaas \
  --sort-by=.metadata.creationTimestamp 
```

Common issues:

* Have you forgotten to create the password required for the gateway?
* The gateway must be able to talk to nats and prometheus. If these are crashing, you probably have networking issues preventing containers from talking to each over or looking up each other over DNS.
* Have you got enough resources free in your cluster for all the services to start? `kubectl describe nodes` or `kubectl top node` should give you some hints here.

### My function isn't updating

Try using the `faas-cli describe` command to check whether the function has been updated.

You can usually view the YAML from Kubernetes for a function with the `kubectl get -n openfaas deploy/NAME` command, then check the logs for the two pods in the gateway and for events in the openfaas-fn namespace.

### My function didn't start

Look for `0/1`, restarts or errors showing up here:

```bash
kubectl get deploy -n openfaas-fn
```

Next, try to describe one of the deployments such as the `nodeinfo`:

```bash
kubectl describe -n openfaas-fn deploy/nodeinfo
```

Next, check the logs for the specific deployment that is failing to start:

```bash
kubectl logs -n openfaas-fn deploy/nodeinfo
```

Next, check the events in the openfaas-fn namespace:

```bash
kubectl get events -n openfaas-fn \
  --sort-by=.metadata.creationTimestamp 
```

Common issues:

* You are using a private registry, but haven't configured any image pull secrets. See: [Kubernetes deployment instructions](https://docs.openfaas.com/deployment/kubernetes/)
* You haven't created a secret which is required for your function to start. Check your function request or stack.yml, and create any missing secrets.
* Your function is crashing due to an error in your code, check the logs.

### My function is timing out

Check the logs of the gateway for signs of a time-out, or non-200 HTTP code:

```bash
kubectl logs -n openfaas deploy/gateway -c gateway
```

Do the same for the provider:

```bash
kubectl logs -n openfaas deploy/gateway -c faas-netes

# Or, if you are using the CRD and Operator:
kubectl logs -n openfaas deploy/gateway -c operator
```

If your function is timing out and you are calling it asynchronously, then check the queue-worker's logs:

```bash
kubectl logs -n openfaas deploy/queue-worker
```

Check the logs of the function

```bash
faas-cli logs NAME
```

Common issues:

* You are using a service mesh, and therefore must set `direct_functions` to `true` so that the gateway uses the name of the function to resolve it
* You have not configured a high enough timeout on all the required components. See [Expanded timeouts](https://docs.openfaas.com/tutorials/expanded-timeouts/)
* You are trying to access the gateway from your function, you must use the string `http://gateway.openfaas:8080`, otherwise it will be unreachable to you.
* Your cloud LoadBalancer may have a timeout set of 60 seconds, which could prevent your call from executing successfully, consider increasing the timeout if you can, or execute the function asynchronously.

### The queue-worker keeps retrying my function

If the queue-worker keeps retrying your function check if `ack_wait` is set to a high enough timeout. It should be set to a value higher than your functions timeout.

See [Expanded timeouts](/tutorials/expanded-timeouts)

### My function gets a nil or empty body

If the body of the HTTP request is empty, the chances are that you are not setting a "Content-type" header in your request to the function.

For example:

```bash
curl -s -i http://127.0.0.1:8080/function/node-tester \
  -H "Content-type: application/json" \
  --data-binary '{"employeeID": 10}'
```

Some legacy HTTP servers such as [WSGI](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface) do not supported the default "chunked" transfer encoding. In this case, if you're using the of-watchdog, you should set the environment variable of `http_buffer_req_body: true`. This causes the HTTP request to be buffered in memory, then sent in one shot to the upstream function.

### My function takes too long to start up

Your function is failing because it takes too long to start-up.

* Learn how to configure a custom HTTP health-check that you can respond to from your function
* Learn how to extend the window between your function starting, and its initial health-check run

See: [Custom HTTP health check](https://docs.openfaas.com/reference/workloads/#custom-http-health-check)

### I want to test my function without deploying it

You can usually do one of the following:

* Use unit-testing in the language of your choice (fastest)
* Use `faas-cli build` followed by `docker run` (easy)
* Try hot-reloading with docker-compose (advanced)

Example:

```bash
faas-cli new --lang go test-this

faas-cli build -f test-this.yml

docker run -p 8081:8080 \
  --rm \
  --name test-this \
  -ti test-this:latest
```

Then invoke your function via `http://127.0.0.1:8081`.

If you need to set environment variable, or to simulate secrets being mounted, you can do so with `--env/-e` and `-v` to simulate mounting secrets at `/var/openfaas/secrets`.

### I am getting an incorrect password error or authorized access

If you're using [ArgoCD to install OpenFaaS](https://www.openfaas.com/blog/bring-gitops-to-your-openfaas-functions-with-argocd/), then it may be changing the password continually whenever it synchronises the app that you created. Make sure you turn off the "generateBasicAuth" setting in values.yaml or the flags you pass.

Create a password for the admin user before you create the ArgoCD App.

```bash
kubectl create secret generic basic-auth \
  -n openfaas \
  --from-literal basic-auth-user=admin \
  --from-literal basic-auth-password=$(openssl rand -base64 32)
```

If you're not an ArgoCD user, make sure that nobody has has reinstalled OpenFaaS, and then check the below on "I forgot my credentials"

In the worst case, restart all the components to force them to reload the password from the Kubernetes secret: `kubectl rollout restart -n openfaas deploy`

### I forgot my credentials for the gateway

Download [arkade](https://arkade.dev).

Now run the following command which will give you the commands you require to retrieve your password.

```bash
arkade info openfaas
```

Alternatively, consider using the [Single-Sign On module from OpenFaaS Pro](https://docs.openfaas.com/reference/authentication/#oidc-and-oauth2-for-the-openfaas-api).

### How do I call one function from another?

See the notes on the asynchronous invocations page under: [Making an asynchronous call from another function](/reference/async)

### Can I manage functions via kubectl?

Yes using the Function Custom Resource: [Learn how to manage your functions with kubectl](https://www.openfaas.com/blog/manage-functions-with-kubectl/)

This is required for Flux or ArgoCD. It also lets you deploy functions via Helm.

### How to I manage different environments like staging and production?

OpenFaaS supports both multiple-namespaces and multiple configuration files for functions. Learn more: [Configure your OpenFaaS functions for staging and production](https://www.openfaas.com/blog/custom-environments/)

### Does OpenFaaS support multi-tenancy?

Yes, if configured correctly. [Call us to find out more](https://openfaas.com/support)

### Traffic is not being spread evenly between functions

You will need to ensure that you are doing one of the following:

* Setting `direct_functions` to `false`, which allows the provider to balance calls randomly between replicas of your functions.
* Use a service mesh like Linkerd or Istio, which can do advanced traffic-management such as least-connections 

### I'm not seeing CPU or RAM data for functions in Grafana with OpenFaaS Pro

You can find the dashboard JSON files in the [Customer GitHub](https://github.com/openfaas/customers). Check that you have the latest version of the dashboard. Sometimes Grafana makes breaking changes in its schema between versions, so edit the panels and check the PromQL statements are present. If they are missing, edit the JSON file in a text editor to retrieve the queries.

You may also want to check that your data source is set to the internal OpenFaaS Prometheus instance.

Finally, check that you've installed the OpenFaaS Pro Helm chart with the ClusterRole setting. This is required to access each node in the cluster to retrieve Pod CPU and RAM usage metrics.

### How can I use structured logs in my function?

By default, the logs will be in the format

```
<RFC8601 Timestamp> <function name> (<container instance>) <msg>
```

By setting the environment variable `prefix_logs` to `false` in your function, this will only send the `<msg>` part to the terminal. This allows you to use structured logs that outputs a JSON (or equivalent) payload.

See the [Logs](https://docs.openfaas.com/cli/logs/#structured-logs) documentation for more details.

### How do I rotate or change a secret that my function is already using?

With either Kubernetes or faasd, you can delete the secret and create it with the new value. Kubernetes also allows you to edit the secret's value using `kubectl`, without deleting it.

Then, with Kubernetes, the secret will be updated on disk without needing to restart the Pod. This change could take several minutes before it reflects.

See also: [Automatic secret updates in Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/#mounted-secrets-are-updated-automatically)

Example:

```bash
faas-cli secret create username \
  --from-literal alex

faas-cli store deploy alpine \
  --secret username \
  --env fprocess="cat /var/openfaas/secrets/username" \
  --name print-secret

echo | faas-cli invoke --name print-secret
alex
```

Then:

```bash
faas-cli secret remove username
faas-cli secret create username \
  --from-literal ellis

# Wait a few minutes
echo | faas-cli invoke --name print-secret
ellis
```

If your handler reads and caches the password in memory, then you'll also need to restart the function's Pod:

```bash
kubectl rollout restart -n openfaas-fn \
  deploy/print-secret
```

### I want to remove OpenFaaS from a cluster

See the [Helm chart instructions](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas)
