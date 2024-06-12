# Multiple Namespaces

!!! info "OpenFaaS For Enterprises"
    This feature is part of [OpenFaaS For Enterprises](/openfaas-pro/introduction).

Multiple namespaces can be used for the following use-cases:

* multi-tenancy (must be combined with other OpenFaaS for Enterprises security features)
* multiple stages or environments within a single cluster with all of dev/staging/prod (to save money)
* logical segregation of groups of functions within a single company or team (ingestion, inference, cron)

!!! warning "Additional configuration required"
    You must configure OpenFaaS to have sufficient permissions to administrate multiple namespaces, this feature will not work in a default installation.

Multiple namespaces are only one part of the configuration required to build a multi-tenant product or service.

Read our guidelines on [How to Build a Multi-tenant Platform with OpenFaaS](https://www.openfaas.com/blog/build-a-multi-tenant-functions-platform/).

## Pre-reqs

* Kubernetes
* OpenFaaS Pro on Kubernetes (or faasd, see final section)

## Configure OpenFaaS with additional permissions

Additional RBAC permissions are required to work with namespaces, using a `ClusterRole` is currently supported for multiple namespaces.

```sh
arkade install openfaas --clusterrole
```

Or via helm, set this override:

```sh
--set clusterRole=true
```

If you cannot use a `ClusterRole` for any reason, then feel free to ask us about [OpenFaaS Pro](https://openfaas.com/support/) which includes a separate configuration suitable for enterprise companies called a "Split Installation".

## Create one or more additional namespaces

A namespace is only considered a function namespace if it has been annotated with `openfaas="1"`. The `kube-system` namespace is considered reserved and cannot be used for functions.

You can create namespaces for OpenFaaS via [its REST API](/reference/rest-api/), with `kubectl`, or your own Kubernetes operator.

### Create a namespace with `kubectl`

```sh
kubectl create namespace staging-fn

kubectl annotate namespace/staging-fn openfaas="1"

kubectl create namespace dev

kubectl annotate namespace/dev openfaas="1"
```

Now you can use `faas-cli` or the API to list namespaces:

```sh
# faas-cli namespaces

Namespaces:
 - dev
 - staging-fn
 - openfaas-fn
```

Deploy some functions:

```sh
# faas-cli store deploy nodeinfo --namespace dev
# faas-cli store deploy nodeinfo --namespace staging-fn
# faas-cli store deploy figlet
```

The URL will be as follows: `http://gateway:port/function/NAME.NAMESPACE`

Use a YAML file:

*stack.yml*

```yaml
provider:
  name: openfaas

functions:
  stronghash:
    namespace: dev
    skip_build: true
    image: functions/alpine:latest
    fprocess: "sha512sum"
```

Deploy the stack.yml and invoke the function:

```sh
faas-cli deploy

openssl rand -base64 32 | faas-cli invoke --namespace dev stronghash
```

Override the namespace configured in the stack.yml when deploying the function and test:

```sh
faas-cli deploy --namespace staging-fn

openssl rand -base64 32 | faas-cli invoke --namespace staging-fn stronghash
```

## Controlling network access

You should consider whether network policies or service mesh policies are required to restrict traffic between namespaces.

## Multi-tenancy

Using an OpenFaaS Pro, you can enable a [Profile](/reference/profiles) to isolate potentially hostile functions using gVisor or Kata containers.

## Namespaces in faasd

For instructions on how to manage multiple namespaces with faasd, see [Serverless For Everyone Else](https://openfaas.gumroad.com/l/serverless-for-everyone-else).
