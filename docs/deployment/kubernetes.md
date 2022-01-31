# Deployment guide for Kubernetes

Before deploying OpenFaaS, you should provision a Kubernetes cluster.

OpenFaaS should run on any compliant Kubernetes distribution.

## Installing OpenFaaS (overview)

There are many options for deploying a local or remote cluster. You can read about the [various Kubernetes distributions here](https://kubernetes.io/docs/setup/).

Once you have a cluster, you can follow the detailed instructions on this page.

* Install the OpenFaaS CLI
* Deploy OpenFaaS using arkade or Helm
* Find your OpenFaaS gateway address
* Retrieve your gateway credentials
* Log in, deploy a function, and try out the UI.

From there, you should consider: adding a TLS certificate with Ingress, switching to the OIDC/OAuth2 plugin for authentication, and tuning-up for production use.

### Options for local cluster clusters

A local cluster is recommended for development and testing, however you can also use managed Kubernetes if you wish.

* [KinD](https://kind.sigs.k8s.io) - upstream Kubernetes in a container with Docker
* [k3d](https://github.com/rancher/k3d) - K3s in a container with Docker
* [k3s](https://k3s.io) - a light-weight Kubernetes distribution ideal for development, edge and IoT
* [minikube](https://minikube.sigs.k8s.io) - the original way to run Kubernetes on your local machine, with a separate Virtual Machine [such as in this guide](https://medium.com/devopslinks/getting-started-with-openfaas-on-minikube-634502c7acdf)
* [microk8s](https://microk8s.io) - a Kubernetes distribution, specifically by Canonical for Ubuntu users.

We recommend using KinD to deploy OpenFaaS, however there are many different ways to run Kubernetes on your machine using a container, or a Virtual Machine.

### Options for remote clusters

Self-hosted Kubernetes:

* [k3sup ("ketchup")](https://k3sup.dev) can be used to build a single or multi-node cluster using cloud VMs

Guides for managed Kubernetes engines:

* [Deploy to Amazon EKS](https://aws.amazon.com/blogs/opensource/deploy-openfaas-aws-eks/)
* [Deploy to Azure AKS](https://docs.microsoft.com/en-us/azure/aks/openfaas)
* [Deploy to DigitalOcean Kubernetes](https://github.com/openfaas/workshop/blob/master/lab1b.md#run-on-digitaloceans-kubernetes-service)
* [Deploy to Google Kubernetes Engine](https://github.com/openfaas/workshop/blob/master/lab1b.md#run-on-gke-google-kubernetes-engine)

### Install the `faas-cli`

!!! tip
    Windows users are encouraged to download [Git Bash](https://git-scm.com/downloads) for use with the OpenFaaS guides and tooling.

You can download the `faas-cli` for MacOS, Windows (Git Bash) or Linux using [arkade](https://arkade.dev/):

```bash
arkade get faas-cli
```

Alternatively, the CLI is available via a download script:

```bash
# MacOS and Linux users

# If you run the script as a normal non-root user then the script
# will download the faas-cli binary to the current folder
$ curl -sL https://cli.openfaas.com | sudo sh

# Windows users with (Git Bash)
$ curl -sL https://cli.openfaas.com | sh
```

Brew is available, however will lag behind by a few versions, so we don't recommend this option.

```bash
brew install faas-cli
```

### Install the OpenFaaS chart using `arkade` or `helm`

There are three recommended ways to install OpenFaaS and you can pick whatever makes sense for you and your team. All options use the OpenFaaS helm chart.

1) Arkade (our recommended option)
    We recommend using [arkade](https://arkade.dev/) to install openfaas, which makes installing OpenFaaS a 1-liner. It still uses Helm, so it can be used in production.

2) Helm
    A helm chart is also available for those who are very well versed with `kubectl` and want to understand exactly what is being installed.

3)  GitOps tooling
    Whilst not recommended for local development, you may want to install OpenFaaS using a GitOps tool in your production environments because these can be configured to keep the chart up to date automatically.

#### 1) Deploy the Chart with `arkade` (fastest option)

The `arkade install` command installs OpenFaaS using its official helm chart. arkade can also install other important software for OpenFaaS users such as `cert-manager` and `nginx-ingress`. It's the easiest and quickest way to get up and running.

You can use [arkade](https://arkade.dev/) to install OpenFaaS to a regular cloud cluster, your laptop, a VM, a Raspberry Pi, or a 64-bit ARM machine.

* Get arkade

  ```sh
  # For MacOS / Linux:
  curl -SLsf https://get.arkade.dev/ | sudo sh

  # For Windows (using Git Bash)
  curl -SLsf https://get.arkade.dev/ | sh
  ```

* Install the OpenFaaS `app`

  ```sh
  arkade install openfaas
  ```

Other options for installation are available with `arkade install openfaas --help`

After the installation you'll receive a command to retrieve your OpenFaaS URL and password.

You can get this information back at any time using:

```bash
arkade info openfaas
```

A good place to go next is the [official training material for OpenFaaS](/tutorials/training/). You'll find links further down on this page.

If you would like to set up public access with a TLS certificate and a custom domain, then follow this tutorial: [Get TLS for OpenFaaS the easy way with arkade](https://blog.alexellis.io/tls-the-easy-way-with-openfaas-and-k3sup/)

#### 2) Deploy the Chart with `helm`

A Helm chart is provided in the `faas-netes` repository.

* [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md)

### 3) GitOps Tooling

There are two popular options for installing the OpenFaaS helm chart with a GitOps approach.

These are advanced tools and are not recommended for local development.

* [Flux v1](https://github.com/fluxcd/flux) / [Flux v2](https://github.com/fluxcd/flux2)
* [ArgoCD](https://argoproj.github.io/argo-cd/)

Refer to the respective documentation for more information.

### Official training

Consult the [official training page](/tutorials/training/) for guides, tutorials, examples and courses.

You can also find a list of [community tutorials, events, and videos](https://github.com/openfaas/faas/blob/master/community.md).

![Preview of the gateway UI](https://camo.githubusercontent.com/72f71cb0b0f6cae1c84f5a40ad57b7a9e389d0b7/68747470733a2f2f7062732e7477696d672e636f6d2f6d656469612f44466b5575483158734141744e4a362e6a70673a6d656469756d)
> Preview of the gateway UI

A walk-through video shows auto-scaling in action and the Prometheus UI: [walk-through video](https://www.youtube.com/watch?v=0DbrLsUvaso).

### Troubleshooting

If you are running into any issues please check out the troubleshooting guide and search the documentation / past issues before raising an issue.

* [Troubleshooting guide](/deployment/troubleshooting/)

### Support 

OpenFaaS Ltd offers support and and a commercial distribution for Production called OpenFaaS Pro.

Find out more by [contacting us](https://openfaas.com/support/)

Guidelines are also provided for [preparing for production](/architecture/production/) and for [performance testing](/architecture/performance).

### Appendix

### Private registries for your functions

See notes here: []()

#### A note for Google Kubernetes Engine (GKE)

You'll need to create an RBAC role with the following command:

```bash
$ kubectl create clusterrolebinding "cluster-admin-$(whoami)" \
  --clusterrole=cluster-admin \
  --user="$(gcloud config get-value core/account)"
```

Also, ensure any [default load-balancer timeouts within GKE](https://cloud.google.com/load-balancing/docs/https/#timeouts_and_retries) are understood and configured appropriately.

#### Deploy with TLS

To enable TLS while using Helm, try one of the following references:

* [Get TLS for OpenFaaS the easy way with arkade](https://blog.alexellis.io/tls-the-easy-way-with-openfaas-and-k3sup/)
* [Configure TLS with nginx-ingress and cert-manager](/reference/ssl/kubernetes-with-cert-manager/)

#### Setting an Image Pull Policy for your functions

Every time a function is deployed or is scaled up, Kubernetes will pull a potentially updated copy of the image from the registry. If you are using static image tags like `latest`, this is necessary.

When set to `IfNotPresent`, function deployments may not be updated when using static image tags like `latest`. `IfNotPresent` is particularly useful when developing locally with minikube. In this case, you can set your local environment to use [minikube's docker](https://minikube.sigs.k8s.io/docs/tasks/docker_daemon/) so `faas-cli build` builds directly into the Docker library used by minikube. `faas-cli push` is unnecessary in this workflow - use faas-cli build then faas-cli deploy.

When set to `Never`, only local (or pulled) images will work. This is useful if you want to tightly control which images are available and run in your Kubernetes cluster.

#### Using Raspberry Pi and ARM

Use `arkade` to install OpenFaaS, it will determine the correct files and container images to install OpenFaaS on an ARM device.

To build and deploy images for Raspberry Pi and ARM, see the notes here: [Building multi-arch images for ARM and Raspberry Pi](/cli/build/)

For a complete tutorial (including OpenFaaS) see:

* Tutorial: [Walk-through — install Kubernetes to your Raspberry Pi in 15 minutes](https://medium.com/p/walk-through-install-kubernetes-to-your-raspberry-pi-in-15-minutes-84a8492dc95a)
* Video: [Kubernetes Homelab with Raspberry Pi 4](https://www.youtube.com/watch?v=qsy1Gwa-J5o)

For the Function Store, use the `--platform` flag to filter to compatible images:

```bash
export OPENFAAS_URL=http://IP:8080/

faas-cli store list --platform armhf
faas-cli store deploy NAME --platform armhf
```

For 64-bit ARM OSes use `--platform arm64` instead.

