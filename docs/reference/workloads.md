# Workloads

OpenFaaS can host multiple types of workloads from functions to microservices, but FaaS Functions have the best support.

## Common properties

All workloads must:

* serve HTTP traffic on TCP port 8080
* assume ephemeral storage
* be stateless

And integrate with a health-check mechanism:

On Swarm:

* create a lock file in `/tmp/.lock` - removing this file signals service degradation
* add a `HEALTHCHECK` instruction if using Docker Swarm

On Kubernetes:

* or enable httpProbe in the `helm` chart and implement `/_/health` as a HTTP endpoint
* create a lock file in `/tmp/.lock` - removing this file signals service degradation

> Note: You can specify a custom HTTP Path for the health-check using the `com.openfaas.health.http.path` annotation

If running in read-only mode, then you can write files to the `/tmp/` mount only. These files may be accessible upon subsequent requests but it is not guaranteed. For instance - if you have two replicas of a function then both may have different contents in their `/tmp/` mount. When running without read-only mode you can write files to the user's home directory subject to the same rules.

### FaaS Functions

To build a FaaS Function simply use the [OpenFaaS CLI](/cli/install.md) to scaffold a new function using one of the official templates or one of your own templates. All FaaS Functions make use of the [OpenFaaS classic watchdog](/architecture/watchdog.md) or the next-gen [of-watchdog](https://github.com/openfaas-incubator/of-watchdog).

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

### Stateless microservices

A stateless microservice can be built using the `dockerfile` language type and the OpenFaaS CLI - or by building a custom Docker image which serves traffic on port `8080` and deploying that via the RESTful API, CLI or UI.

An example of a stateless microservice may be an Express.js application using Node.js, a Sinatra app with Ruby or an ASP.NET 2.0 Core web-site.

Use of the [OpenFaaS next-gen of-watchdog](https://github.com/openfaas-incubator/of-watchdog) is optional, but recommended for stateless microservices to provide a consistent experience for timeouts, logging and configuration.

On Kubernetes is possible to run any container image as an OpenFaaS function as long as your application exposes port 8080 and has a HTTP health check endpoint.

You can specify the HTTP path of your health check and the initial check delay duration with the following annotations:

* `com.openfaas.health.http.path`
* `om.openfaas.health.http.initialDelay`

Stack file example:

```yaml
functions:
  kubesec:
    image: docker.io/stefanprodan/kubesec:v2.1
    skip_build: true
    annotations:
      com.openfaas.health.http.path: "/healthz"
      com.openfaas.health.http.initialDelay: "30s"
``` 

> Note: The initial delay value must be a valid Go duration e.g. `80s` or `3m`. 



