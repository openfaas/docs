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

Optional components include event connectors, such as: the Kafka connector, SQS, SNS, and PostgreSQL connector. You can find details for each in the OpenFaaS Pro docs referenced above.

For OpenFaaS for Enterprises:

* Multiple namespaces are automatically detected and enabled, when you set `clusterRole: true` in the chart.
* The Function Builder has its own chart, which is installed separately.
* IAM and SSO can be configured up front, or later down the line when you have the base installation complete

You'll also want to install our Grafana dashboards, which give you access to additional metrics:

* [Grafana Dashboard JSON files in the Customer Community](https://github.com/openfaas/customers/tree/master/dashboards)

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
  authorization:
    enabled: true
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

The recommended values.yaml file enables NATS authentication. If you are upgrading from OpenFaaS CE or enabling NATS authentication for the first time on an existing installation an authorization token secret should be created.

If this is your first time installing OpenFaaS Pro you can ignore this step. The Helm Chart will generate the secret automatically.

Create a secret for the NATS authorization token:

```sh
# openssl is preferred to generate a random secret:
openssl rand -base64 32 > ./nats-token

kubectl create secret generic \
    -n openfaas \
    nats-token \
    --from-file token=./nats-token
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
helm repo update && \
  helm upgrade --install openfaas \
  --install openfaas/openfaas \
  --namespace openfaas \
  -f ./values-custom.yaml
```

It is essential that you keep hold of any custom values.yaml files that you create or use during installation. These are needed in order to receive support from OpenFaaS Ltd and to upgrade your installation.

### Leader election

Each OpenFaaS gateway deployment contains both the gateway and the operator. The operator is responsible for managing the Function CRD, and creating Kubernetes resources such as Services and Deployments in response to changes in the Function CRD. If you are running more than one replica of the gateway deployment, then you must enable leaderElection to avoid conflicts and warnings between the operator instances.

### What happens if you disable the ClusterRole?

The ClusterRole is required for

* OpenFaaS Standard and for Enterprises for Prometheus in order to discover and scrape node-level metrics for monitoring, API metrics and CPU-based autoscaling
* OpenFaaS for Enterprises to manage functions in various namespaces

## How to upgrade OpenFaaS

OpenFaaS Standard and For Enterprises are both installed and upgraded in the same way, so you use the same `helm upgrade --install` command from the second above. There is no reason to "uninstall" the helm chart to perform an upgrade.

The only time that `helm upgrade --install` may not work is when you are changing from `clusterRole: false` to `clusterRole: true`. In this instance, you will need to delete the conflicting Kubernetes objects one by one as directed by the output from helm, before running `helm upgrade --install` again.

We recommend upgrading OpenFaaS every 1-2 weeks to ensure you have the latest fixes, security patches and features. Breaking changes are rare, and are published in the [Customer Community](https://github.com/openfaas/customers).

### Automatic updates with ArgoCD or FluxCD

You can use ArgoCD or FluxCD to manage the installation of OpenFaaS by providing a custom values.yaml file. In this way, newer versions of the OpenFaaS components and Helm chart will be applied automatically, as they become available.

See also:

* [Bring GitOps to your OpenFaaS functions with ArgoCD](https://www.openfaas.com/blog/bring-gitops-to-your-openfaas-functions-with-argocd/)
* [Upgrade to Flux v2 to keep OpenFaaS up to date](https://www.openfaas.com/blog/upgrade-to-fluxv2-openfaas/)

### Installing new Custom Resource Definitions (CRDs)

CRDs are split into two categories:

1. Those that are bundled via the chart and updated on each installation/update - including Function, Profile and FunctionIngress. These are kept as templates so that they are updated whenever the chart is updated.
2. Those which are required before the chart is installed for IAM policies. The second category is required at installation time due to installing instances of these Custom Resources at installation time.

To update the type 2 CRDs, you'll need to run the [steps under "2.1 CRDs in the ./crds folder
"](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/crds/README.md).

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

### Dedicated queues and queue-workers

There is a separate Helm chart that you can install to provision and manage queue-workers for dedicated queues.

Dedicated queues are useful tenant-level isolation, or to provide a higher Quality of Service (QoS) for certain functions.

Learn more:

* [JetStream queue-worker](/openfaas-pro/jetstream/#multiple-queues)
* [Dedicated JetStream queue-worker helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/queue-worker)

### OpenFaaS Pro CLI

The OpenFaaS Pro CLI provides additional functionality on top of faas-cli, such as build-time secrets, and a `local-run` command to try out functions without deploying them.

```bash
faas-cli plugin get pro
faas-cli pro enable
```

You'll need a GitHub account in your company's GitHub organisation to use this feature. If you cannot get one for some reason, or use GitLab, please let us know and we'll provide an alternative mechanism to activate the CLI.

See also: [faas-cli installation](/cli/install)

### Function Builder API

The OpenFaaS Pro Function Builder API can be deployed through a [separate Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder).

[View chart](/openfaas-pro/builder)

### Identity and Access Management (IAM) and Single-Sign On (SSO)

Identity and Access Management (IAM) and Single-Sign On (SSO) are closely related and are often configured at the same time. IAM is how Identity Providers (IdPs) are configured and how users are authenticated and authorised. Single-Sign On is how users leverage an IdP to authenticate and authorise themselves to the OpenFaaS dashboard, API and CLI.

* [Identity and Access Management (IAM)](/openfaas-pro/iam/overview)
* [Single-Sign On (SSO)](/openfaas-pro/sso/overview)
