# Workloads

OpenFaaS can host multiple types of workloads from functions to microservices, but FaaS Functions have the best support.

## Common properties

All workloads must:

* serve HTTP traffic on TCP port 8080
* assume ephemeral storage
* be stateless

And integrate with a health-check mechanism with Kubernetes:

* or enable httpProbe in the `helm` chart and implement `/_/health` as a HTTP endpoint
* create a lock file in `/tmp/.lock` - removing this file signals service degradation

> Note: You can specify a custom HTTP Path for the health-check using the `com.openfaas.health.http.path` annotation

If running in read-only mode, then you can write files to the `/tmp/` mount only. These files may be accessible upon subsequent requests but it is not guaranteed. For instance - if you have two replicas of a function then both may have different contents in their `/tmp/` mount. When running without read-only mode you can write files to the user's home directory subject to the same rules.

### Functions

To build a function simply use the [OpenFaaS CLI](/cli/install) to scaffold a new function using one of the official templates or one of your own templates. All FaaS Functions make use of the [OpenFaaS classic watchdog](/architecture/watchdog) or the next-gen [of-watchdog](https://github.com/openfaas/of-watchdog).

```
faas-cli template pull
faas-cli new --list
```

Or build your own templates Git repository and then pass that as an argument to `faas-cli template pull`

```
faas-cli template pull https://github.com/my-org/templates
faas-cli new --list
```

Custom binaries can also be used as a function. Just use the `dockerfile` language template and replace the `fprocess` variable with the command you want to run per request. If you would like to pipe arguments to a CLI utility you can prefix the command with `xargs`.

### Running an existing Docker image on OpenFaaS

Let's take a Node.js app which listens on traffic using port 3000, and assume that we can't make any changes to it.

You can view its Dockerfile and code at: [alexellis/expressjs-k8s](https://github.com/alexellis/expressjs-k8s/) and the image is published to the Docker Hub at: `alexellis2/service:0.3.6`

Start by creating a new folder:

```bash
mkdir -p node-service/
```

Write a custom Dockerfile `./node-service/Dockerfile`:

```Dockerfile
# Import the OpenFaaS of-watchdog
FROM openfaas/of-watchdog:0.7.2 as watchdog

# Add a FROM line from your existing image
FROM alexellis2/service:0.3.6

# Let's say that the image listens on port 3000 and 
# that we can't change that easily
ENV http_port 3000

# Install the watchdog from the base image
COPY --from=watchdog /fwatchdog /usr/bin/

# Now set the watchdog as the start-up process
# Along with the HTTP mode, and an upstream URL to 
# where your HTTP server will be running from the original
# image.
ENV mode="http"
ENV upstream_url="http://127.0.0.1:3000"

# Set fprocess to the value you have in your base image
ENV fprocess="node index.js"
CMD ["fwatchdog"]
```

Now create a stack.yml at the root directory `./stack.yml`:

```yaml
provider:
  name: openfaas
functions:
  node-service:
    handler: ./node-service
    image: docker.io/alexellis2/node-service-watchdog:0.1.0
    lang: dockerfile
```

Now run `faas-cli up`

Your code will now listen on port 8080 and fulfil the serverless definition including automatic health-checks and a graceful shutdown.

You can then access the service at: `http://127.0.0.1:8080/function/node-service`

### Custom service account

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

When using Kubernetes, OpenFaaS Pro functions can assume a ServiceAccount in the namespace in which they are deployed.

For example if a workload needed to read logs from the Kubernetes API using a ServiceAccount named `function-logs-sa`, you could bind it in this way:

*stack.yml*

```yaml
functions:
  system-logs:
     annotations:
       com.openfaas.serviceaccount: function-logs-sa
```

Here is an example `Role` that can list pods and work with Pod logs within the `openfaas-fn` namespace:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: function-logs-role
  namespace: openfaas-fn
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "create"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: function-logs-role-binding
  namespace: openfaas-fn
subjects:
- kind: ServiceAccount
  name: function-logs-sa
  namespace: openfaas-fn
roleRef:
  kind: Role
  name: function-logs-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: function-logs-sa
  namespace: openfaas-fn
  labels:
    app: openfaas
```

### Custom TerminationGracePeriod for long running functions

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

You can configure your functions to drain any requests in flight when scaling down. This prevents errors and makes sure all work is processed, before Kubernetes finally removes any Pods.

To set a custom [TerminationGracePeriod](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) for a function, configure a `write_timeout` environment variable.

```yaml
functions:
  email-batch:
     environment:
       write_timeout: "1m"
```

When scaling down the function after scaling up, or scaling to zero, Kubernetes will wait for 1m before removing the function. If there is no work to be done, it could exit sooner because the OpenFaaS watchdog does a safe shutdown.

Read more here: [Improving long-running jobs for OpenFaaS users](https://www.openfaas.com/blog/long-running-jobs/)

### Stateless microservices

A stateless microservice can be built using the `dockerfile` language type and the OpenFaaS CLI - or by building a custom Docker image which serves traffic on port `8080` and deploying that via the RESTful API, CLI or UI.

An example of a stateless microservice may be an Express.js application using Node.js, a Sinatra app with Ruby or an ASP.NET 2.0 Core website.

Use of the [OpenFaaS next-gen of-watchdog](https://github.com/openfaas/of-watchdog) is optional, but recommended for stateless microservices to provide a consistent experience for timeouts, logging and configuration.

On Kubernetes is possible to run any container image as an OpenFaaS function as long as your application exposes port 8080 and has a HTTP health check endpoint.

#### Custom HTTP health checks

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

You can specify the HTTP path of your health check and the initial check delay duration with the following annotations:

* `com.openfaas.health.http.path`
* `com.openfaas.health.http.initialDelay`
* `com.openfaas.health.http.periodSeconds`

For example, you may have a function that takes 30s to initialise, but then only needs to be checked every 5s after that.

```yaml
functions:
  kubesec:
    image: docker.io/stefanprodan/kubesec:v2.1
    skip_build: true
    annotations:
      com.openfaas.health.http.path: "/healthz"
      com.openfaas.health.http.initialDelay: "30s"
      com.openfaas.health.http.periodSeconds: 5s
``` 

> Note: The initial delay value must be a valid Go duration e.g. `80s` or `3m`.

Readiness probes use the same HTTP path as the health check by default. A custom ready path can be set with the following annotation:

* `com.openfaas.ready.http.path`

The `timeoutSeconds` value for liveness and readiness probes can be set globally for the installation: [OpenFaaS chart reference](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#faas-netes--operator). A global `initialDelaySeconds` and `periodSeconds` can also be set for the installation, any overrides set for these two values in annotations will take precedence.
