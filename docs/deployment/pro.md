# Install OpenFaaS Pro

There are a number of components that make up the two editions of OpenFaaS Pro: OpenFaaS Standard and OpenFaaS for Enterprises.

The reference documentation for each feature is available here: [Docs: OpenFaaS Pro](https://docs.openfaas.com/openfaas-pro/introduction/)

The Core platform is installed by switching the `openfaasPro` flag to `true` in the openfaas chart, however there are extra components that you will likely need or want to customise.

## Core platform

The Core platform features of OpenFaaS Standard and OpenFaaS for Enterprises consists of:

* OpenFaaS Pro Gateway
* OpenFaaS Pro Kubernetes Operator with Custom Resource Definitions (CRDs) i.e. `Function` and `Profile`
* OpenFaaS Pro UI Dashboard
* OpenFaaS Pro Autoscaler (including Scale to Zero)
* OpenFaaS Pro Queue Worker for JetStream
* OpenFaaS Pro CLI `faas-cli pro`

The Core platform is installed using the same Helm chart as OpenFaaS CE, only with some additional values set, to deploy the additional Pro components and versions.

For OpenFaaS for Enterprises:

* Multiple namespaces are automatically detected and enabled, when you set `clusterRole: true` in the chart.
* The Function Builder has its own chart, which is installed separately.
* IAM and SSO can be configured up front, or later down the line when you have the base installation complete

You'll also want to install our Grafana dashboards, which give you access to additional metrics:

* [Grafana Dashboards](https://github.com/openfaas/customers/tree/master/dashboards)

## Need to update your license?

The easiest way to do this is to delete the existing secret, create a new one, then restart all the deployments in the `openfaas` namespace.

```sh
kubectl delete secret -n openfaas \
  openfaas-license

kubectl create secret generic \
  -n openfaas \
  openfaas-license \
  --from-file license=$HOME/.openfaas/LICENSE

kubectl rollout restart deployment -n openfaas
```

## Installation

Detailed installation instructions including the various options for values.yaml are available in the [`openfaas` helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

We recommend using the following configuration for the `values-custom.yaml` file:

```yaml
openfaasPro: true
clusterRole: true

operator:
  create: true
  leaderElection:
    enabled: true

gateway:
  replicas: 3

  # 10 minute timeout
  upstreamTimeout: 10m
  writeTimeout: 10m2s
  readTimeout: 10m2s

autoscaler:
  enabled: true

dashboard:
  enabled: true

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

helm upgrade --install openfaas \
  --install openfaas/openfaas \
  --namespace openfaas \
  -f ./values-custom.yaml
```

It is essential that you keep hold of any custom values.yaml files that you create or use during installation. These are needed in order to receive support from OpenFaaS Ltd and to upgrade your installation.

## How to upgrade OpenFaaS

!!! Warning "Do not uninstall the Helm chart"

    Beware that the Helm chart should never be uninstalled, and if you do run "helm uninstall" the Function Custom Resource Definition (CRD), and all Functions will be removed from the cluster. This is standard behaviour for Kubernetes when requesting an "uninstallation".

OpenFaaS Standard and For Enterprises are both installed and upgraded in the same way, so you use the same `helm upgrade --install` command from the second above.

The only time that `helm upgrade --install` may not work is when you are changing from `clusterRole: false` to `clusterRole: true`. In this instance, you will need to delete the conflicting Kubernetes objects one by one as directed by the output from helm, before running `helm upgrade --install` again.

### Installing new Custom Resource Definitions (CRDs)

CRDs are only installed with the initial installation of OpenFaaS, therefore, if new ones have been added, or updated, you'll need to extract them from the helm chart and apply them manually.

If you're upgrading to OpenFaaS IAM for instance, you can generate the CRDs, and then apply the files in the `/tmp/openfaas/crds` folder:

```sh
helm template openfaas/openfaas \
  --include-crds=true \
  --output-dir=/tmp

...

wrote /tmp/openfaas/crds/iam.openfaas.com_jwtissuers.yaml
wrote /tmp/openfaas/crds/iam.openfaas.com_policies.yaml
wrote /tmp/openfaas/crds/iam.openfaas.com_roles.yaml
```

Then:

```sh
kubectl apply -f /tmp/openfaas/crds
```

### Automatic upgrades with ArgoCD or FluxCD

You can use ArgoCD or FluxCD to manage the installation of OpenFaaS by providing a custom values.yaml file. In this way, newer versions of the OpenFaaS components and Helm chart will be applied automatically, as they become available.

See also:

* [Bring GitOps to your OpenFaaS functions with ArgoCD](https://www.openfaas.com/blog/bring-gitops-to-your-openfaas-functions-with-argocd/)
* [Upgrade to Flux v2 to keep OpenFaaS up to date](https://www.openfaas.com/blog/upgrade-to-fluxv2-openfaas/)

## How to check your installation

We recommend running the OpenFaaS config checker after installation is complete. This will help verify that your installation is correct.

See also: [OpenFaaS config checker](https://github.com/openfaas/config-checker)

## Additional components

### Grafana dashboards

You should install the various [Grafana Dashboards](https://github.com/openfaas/customers/tree/master/dashboards) offered to OpenFaaS customers for comprehensive monitoring of your OpenFaaS installation.

### External NATS cluster (optional)

The version of NATS installed by the OpenFaaS chart is suitable for light use in production, and for development and staging. For critical use, we recommend installing the NATS helm chart, with multiple replicas, and/or a Persistent Volume (PV).

If you leave NATS as it is, and the NATS container crashes, or the Kubernetes node where it is running is removed, you will lose all asynchronous function invocations in the queue which have not yet been processed.

See full instructions in the [Customer Community](https://github.com/openfaas/customers/)

### Event connectors

There are a number of event connectors for OpenFaaS, all of which are installed separately as required using their respective Helm charts.

[View helm charts for event connectors](https://github.com/openfaas/faas-netes/tree/master/chart)

### OpenFaaS Pro CLI

The OpenFaaS Pro CLI provides additional functionality on top of faas-cli, such as build-time secrets, and a `local-run` command to try out functions without deploying them.

```bash
faas-cli plugin get pro
faas-cli pro enable
```

You'll need a GitHub account in your company's GitHub organisation to use this feature. If you cannot get one for some reason, or use GitLab, please let us know and we'll provide an alternative mechanism to activate the CLI.

See also: [faas-cli installation](/cli/install)

### Function Builder API

The OpenFaaS Pro Function Builder API can be deployed through a separate Helm chart.

[View chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder)

### SSO & IAM

Single-Sign On and IAM are closely related and are often configured at the same time.

See our walkthrough for an overview of how this works: [Walkthrough of Identity and Access Management (IAM) for OpenFaaS](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/).
