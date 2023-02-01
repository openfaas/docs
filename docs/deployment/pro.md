# Install OpenFaaS Pro

There are a number of components that make up OpenFaaS Pro and OpenFaaS for Enterprises.

The reference documentation for each feature is available here: [Docs: OpenFaaS Pro](https://docs.openfaas.com/openfaas-pro/introduction/)

The Core platform is installed by switching the `openfaasPro` flag to `true` in the openfaas chart, however there are extra components that you will likely need or want to customise.

## Core platform

The Core platform features of OpenFaaS Pro consists of:

* OpenFaaS Pro Gateway
* OpenFaaS Pro Kubernetes Operator with Custom Resource Definitions (CRDs) i.e. `Function` and `Profile`
* OpenFaaS Pro UI Dashboard
* OpenFaaS Pro Autoscaler
* OpenFaaS Pro Queue Worker for JetStream
* OpenFaaS Pro CLI `faas-cli pro`

The Core platform is installed using the same Helm chart as OpenFaaS CE, only with some additional values set, to deploy the additional Pro components and versions.

You'll also want to install our Grafana dashboards, which give you access to additional metrics:

* [Grafana Dashboards](https://github.com/openfaas/openfaas-pro/tree/master/dashboards)

## Pro CLI

The OpenFaaS Pro CLI provides additional functionality on top of faas-cli, such as build-time secrets, and a `local-run` command to try out functions without deploying them.

```bash
faas-cli plugin get pro
faas-cli pro --help
```

See also: [faas-cli installation](/cli/install)

## Event connectors

There are a number of event connectors for OpenFaaS, all of which are installed separately as required using their respective Helm charts.

[View helm charts for event connectors](https://github.com/openfaas/faas-netes/tree/master/chart)

## Function Builder API

The OpenFaaS Pro Function Builder API can be deployed through a separate Helm chart.

[View chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder)

## Installation

Detailed installation instructions including the various options for values.yaml are available in the [`openfaas` helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

We recommend using the following configuration for the `values.yaml` file:

```yaml
openfaasPro: true

operator:
  create: true

clusterRole: true

gateway:
  replicas: 3

queueWorker:
  replicas: 3

queueWorkerPro:
  maxInflight: 50

queueMode: jetstream

nats:
  streamReplication: 1
```

You can find explanations for each configuration item in the [values-pro.yaml](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values-pro.yaml) file on GitHub.

Create the namespaces for OpenFaaS Pro and its functions:

```sh
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```

Create a secret for your OpenFaaS license:

```sh
kubectl create secret generic \
  -n openfaas \
  openfaas-license \
  --from-file license=$HOME/.openfaas/LICENSE
```

Add the OpenFaaS helm chart repo:

```sh
helm repo add openfaas https://openfaas.github.io/faas-netes/
```

Next, create the required tokens for the OpenFaaS Pro UI Dashboard:

[OpenFaaS Pro dashboard pre-requisites](https://docs.openfaas.com/openfaas-pro/dashboard/#installation)

If this is your first time installing the dashboard, we recommend using "localhost" in the `publicURL` field, then you can access the dashboard through port-forwarding.

Then add the Helm chart repo, update it and deploy the chart, running `helm repo update` before each upgrade, to make sure you have the latest versions.

```sh
helm repo update

helm upgrade openfaas \
    --install chart/openfaas \
    --namespace openfaas \
    -f ./values-pro.yaml
```

## How to check your installation

We recommend running the OpenFaaS config checker after installation is complete. This will help verify that your installation is correct.

See also: [OpenFaaS config checker](https://github.com/openfaas/config-checker)
