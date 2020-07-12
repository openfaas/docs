# Namespaces support

OpenFaaS has support for multiple-namespaces, this is currently in Alpha and is being [worked on actively and tracked on this issue](https://github.com/openfaas/faas-netes/issues/511).

Multiple namespaces can be used for the following use-cases:

* multi-tenancy (when combined with NetworkPolicy)
* multiple stages or environments within a single cluster - i.e. dev/staging/prod
* logical segregation of stacks within a single company or team

## Pre-reqs

* Kubernetes
* faas-netes for the faas-provider (default)

## Configure OpenFaaS with additional permissions

This is only implemented for faas-netes at present, the default Kubernetes provider.

Additional RBAC permissions are required to work with namespaces, therefore you have two options:

* 1) simplest option - use a ClusterRole

    ```sh
    arkade install openfaas --clusterrole
    ```

    Or via helm, set this override:

    ```
    --set clusterRole=true
    ```

    The ClusterRole will allow OpenFaaS to operate in each namespace you create.


* 2) Least-privilege option

    Install as per normal:

    ```sh
    arkade install openfaas
    ```

    Then add a Role and RoleBinding for each additional namespace, use the example from the default `openfaas-fn` namespace.

## Create one or more additional namespaces

Each additional function namespace must be annotated, `kube-system` is black-listed and not available at this time.

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

head -c 16 /dev/urandom | faas-cli invoke --namespace dev stronghash
```

Override the namespace configured in the stack.yml when deploying the function and test:

```sh
faas-cli deploy --namespace staging-fn

head -c 16 /dev/urandom | faas-cli invoke --namespace staging-fn stronghash
```
