# Deployment guide for Kubernetes

Before deploying OpenFaaS, you should provision a Kubernetes cluster.

## Installing OpenFaaS (an overview)

There are many options for deploying a local or remote cluster. You can read about the [various Kubernetes distributions here](https://kubernetes.io/docs/setup/).

Once you have a cluster, you can follow the detailed instructions on this page.

* Install OpenFaaS CLI
* Deploy OpenFaaS using via helm or arkade
* Find your OpenFaaS gateway address
* Retrieve your gateway credentials
* Log in, deploy a function, and try out the UI.

From there, you should consider: adding a TLS certificate with Ingress, switching to the OIDC/OAuth2 plugin for authentication, and tuning-up for production use.

## Build your cluster

### Local clusters

Below are the most popular ways to run a local Kubernetes cluster, but OpenFaaS should run on any.

* [k3d](https://github.com/rancher/k3d) - makes k3s available on any computer where Docker is also running
* [KinD](https://kind.sigs.k8s.io) - upstream Kubernetes running inside a Docker container.
* [k3s](https://k3s.io) - a light-weight Kubernetes distribution ideal for edge and development - compatible with Raspberry Pi & ARM64 (Equinix Metal, AWS Graviton, etc)
* [minikube](https://minikube.sigs.k8s.io) - a popular, but heavy-weight option that creates a Linux virtual machine your computer using VirtualBox or similar
* [microk8s](https://microk8s.io) - a Kubernetes distribution, specifically for Ubuntu users.

### Remote/managed options

You can run `k3s` and `k3d` on a single node Virtual Machine so that you don't have to run Kubernetes on your own computer.

* The [k3sup ("ketchup")](https://k3sup.dev) tool can help you to do this by installing k3s onto a remote VM

Kubernetes services/engines:

* [Deploy to DigitalOcean Kubernetes](https://github.com/openfaas/workshop/blob/master/lab1b.md#run-on-digitaloceans-kubernetes-service)
* [Deploy to Google Kubernetes Engine](https://github.com/openfaas/workshop/blob/master/lab1b.md#run-on-gke-google-kubernetes-engine)
* [Deploy to Amazon EKS](https://aws.amazon.com/blogs/opensource/deploy-openfaas-aws-eks/)
* [Deploy to Azure AKS](https://docs.microsoft.com/en-us/azure/aks/openfaas)

A guide is available for configuring minikube here:

* [Getting started with OpenFaaS on minikube](https://medium.com/devopslinks/getting-started-with-openfaas-on-minikube-634502c7acdf)

!!! tip
    Are you using Google Kubernetes Engine (GKE)? You'll need to create an RBAC role with the following command:

    ```bash
    $ kubectl create clusterrolebinding "cluster-admin-$(whoami)" \
      --clusterrole=cluster-admin \
      --user="$(gcloud config get-value core/account)"
    ```

    Also, ensure any [default load-balancer timeouts within GKE](https://cloud.google.com/load-balancing/docs/https/#timeouts_and_retries) are understood and configured appropriately.

### Install the `faas-cli`

Windows users are encouraged to download [Git Bash](https://git-scm.com/downloads) for use with the OpenFaaS guides and tooling.

You can install the OpenFaaS CLI using `curl` on MacOS, Windows (Git Bash) and Linux.

```bash
# MacOS and Linux users

# If you run the script as a normal non-root user then the script
# will download the faas-cli binary to the current folder
$ curl -sL https://cli.openfaas.com | sudo sh

# Windows users with (Git Bash)
$ curl -sL https://cli.openfaas.com | sh

```

The CLI is also available on `brew` for MacOS users, however it may lag behind by a few releases:

```bash
brew install faas-cli
```

### Install the OpenFaaS chart using `arkade` or `helm`

There are three recommended ways to install OpenFaaS and you can pick whatever makes sense for you and your team.

1) Helm with `arkade install` - arkade installs OpenFaaS to Kubernetes using its official helm chart and is the easiest and quickest way to get up and running.
2) `helm` client - sane defaults and easy to configure through YAML or CLI flags. Secure options such as `helm template` or `helm 3` also exist for those working within restrictive environments.
3) With GitOps tooling. You can install OpenFaaS and keep it up to date with [Flux](https://github.com/fluxcd/flux) or [ArgoCD](https://argoproj.github.io/argo-cd/).

#### 1) Deploy the Chart with `arkade` (fastest option)

The `arkade install` command installs OpenFaaS using its official helm chart, but without using `tiller`, a [component which is insecure by default](https://engineering.bitnami.com/articles/running-helm-in-production.html). arkade can also install other important software for OpenFaaS users such as `cert-manager` and `nginx-ingress`. It's the easiest and quickest way to get up and running.

You can use arkade to install OpenFaaS to a regular cloud cluster, your laptop, a VM, a Raspberry Pi, or a 64-bit ARM machine.

* Get arkade

```sh
# For MacOS / Linux:
curl -SLsf https://dl.get-arkade.dev/ | sudo sh

# For Windows (using Git Bash)
curl -SLsf https://dl.get-arkade.dev/ | sh
```

* Install the OpenFaaS `app`

If you're using a managed cloud Kubernetes service which supplies LoadBalancers, then run the following:

```sh
arkade install openfaas --load-balancer
```

> Note: the `--load-balancer` flag has a default of `false`, so by passing the flag, the installation will request one from your cloud provider.

If you're using a local Kubernetes cluster or a VM, then run:

```sh
arkade install openfaas
```

After the installation you'll receive a command to retrieve your OpenFaaS URL and password.

Other options for installation are available with `arkade install openfaas --help`

For cloud users run `kubectl get -n openfaas svc/gateway-external` and look for `EXTERNAL-IP`. This is your gateway address.

#### 2) Deploy the Chart with `helm`

A Helm chart is provided in the `faas-netes` repository. Follow the link below then come back to this page.

* [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md)

    !!! note
        Some users may have concerns about using helm charts due to [security concerns with the `tiller` component](https://engineering.bitnami.com/articles/running-helm-in-production.html). If you fall into this category of users, then don't worry, you can still benefit from the helm chart without using `tiller`.
        
        See the [Chart readme](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md#deployment-with-helm-template) for how to generate your own static YAML files using `helm template`.

#### Notes for Raspberry Pi & 32-bit ARM (armhf)

Use `arkade` to install OpenFaaS, it will determine the correct files to use to install OpenFaaS.

For a complete tutorial (including OpenFaaS) see:

* Tutorial: [Walk-through — install Kubernetes to your Raspberry Pi in 15 minutes](https://medium.com/p/walk-through-install-kubernetes-to-your-raspberry-pi-in-15-minutes-84a8492dc95a)
* Video: [Kubernetes Homelab with Raspberry Pi 4](https://www.youtube.com/watch?v=qsy1Gwa-J5o)

When creating new functions you will need to run the build on an armhf host.

> Note: expert users can create or use [multi-arch templates](https://github.com/alexellis/multiarch-templates) which can build on a PC and deploy to an armhf host.

* You can run `faas-cli deploy` from any computer using `--gateway` or `OPENFAAS_GATEWAY`
* But you must build Docker images on a Raspberry Pi, not on your PC or laptop. 

For the Function Store, use the following:

```bash
faas-cli store list --platform armhf
faas-cli store deploy NAME
```

Instructions are almost identical for ARM64 users, but use `--platform arm64` instead.

### Getting help, expert installations and proof-of-concepts 

* You can get help by connecting with the community on the [Community Page](/community/).
* OpenFaaS Ltd offers expert installation, proof-of-concepts, and architecture reviews. Get in touch at: [sales@openfaas.com](mailto:sales@openfaas.com) to find out more.
* The [OpenFaaS Premium Subscription](https://openfaas.com/support/) offers enterprise-grade authentication with SSO and OpenID Connect (OIDC).
* Guidelines are also provided for [preparing for production](/architecture/production/) and for [performance testing](/architecture/performance).

#### Learn the OpenFaaS fundamentals

The community has built a workshop with 12 self-paced hands-on labs. Use the workshop to begin learning OpenFaaS at your own pace:

* [OpenFaaS workshop](/tutorials/workshop/)

You can also find a list of [community tutorials, events, and videos](https://github.com/openfaas/faas/blob/master/community.md).

![](https://camo.githubusercontent.com/72f71cb0b0f6cae1c84f5a40ad57b7a9e389d0b7/68747470733a2f2f7062732e7477696d672e636f6d2f6d656469612f44466b5575483158734141744e4a362e6a70673a6d656469756d)

A walk-through video shows auto-scaling in action and the Prometheus UI: [walk-through video](https://www.youtube.com/watch?v=0DbrLsUvaso).

### Troubleshooting

If you are running into any issues please check out the troubleshooting guide and search the documentation / past issues before raising an issue.

* [Troubleshooting guide](/deployment/troubleshooting/)

### Advanced

This section covers additional advanced topics beyond the initial deployment.

#### Deploy with TLS

To enable TLS while using Helm, try one of the following references:

* [Using nginx-ingress and cert-manager](/reference/ssl/kubernetes-with-cert-manager/)

#### Use a private registry with Kubernetes

If you are using a hosted private Docker registry ([Docker Hub](https://hub.docker.com/), or other),
in order to check how to configure it, please visit the Kubernetes [documentation](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry).

If you try to deploy using `faas-cli deploy` it will fail because the Kubernetes kubelet component will not have credentials to authorize the docker image pull request.

Once you have pushed an image to a private registry using `faas-cli push` follow the instructions below to either create a pull secret that can be referenced by each function which needs it, or create a secret for the ServiceAccount in the `openfaas-fn` namespace so that any functions which need it can make use of it.

If you need to troubleshoot the use of a private image then see the Kubernetes section of the [troubleshooting guide](./troubleshooting.md).

You can set up your own private Docker registry using this tutorial: [Get a TLS-enabled Docker registry in 5 minutes](https://blog.alexellis.io/get-a-tls-enabled-docker-registry-in-5-minutes/)

##### Option 1 - use an ad-hoc image pull secret

To deploy your function(s) first you need to create an [Image Pull Secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) with the commands below.

Setup some environmental variables:

```bash
export DOCKER_USERNAME=<your_docker_username>
export DOCKER_PASSWORD=<your_docker_password>
export DOCKER_EMAIL=<your_docker_email>
```

Then run this command to create the secret:

```bash
$ kubectl create secret docker-registry dockerhub \
    -n openfaas-fn \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-email=$DOCKER_EMAIL
```

> Note if not using the Docker Hub you will also need to pass `--docker-server` and the address of your remote registry.

The secret *must* be created in the `openfaas-fn` namespace or the equivalent if you have customised this.

Create a sample function with a `--prefix` variable:

```sh
faas-cli new --lang go private-fn --prefix=registry:port/repo
mv private-fn.yml stack.yml
```

Update the `stack.yml` file and add a reference to the new secret:

```yml
secrets:
      - dockerhub
```

Now deploy the function using `faas-cli up`.

##### Option 2 - Link an image pull secret to the namespace's ServiceAccount

Rather than specifying the pull secret for each function that needs it you can bind the secret to the namespace's ServiceAccount. With this option you do not need to update the `secrets:` section of the `stack.yml` file.

Create the image pull secret in the `openfaas-fn` namespace (or equivalent):

```bash
$ kubectl create secret docker-registry my-private-repo \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-email=$DOCKER_EMAIL \
    --namespace openfaas-fn
```

If needed, pass in the `--docker-server` address.

Use the following command to edit the default ServiceAccount's configuration:

```sh
$ kubectl edit serviceaccount default -n openfaas-fn
```

At the bottom of the manifest add:

``` yaml
imagePullSecrets:
- name: my-private-repo
```

Save the changes in the editor and this configuration will be applied.

The OpenFaaS controller will now deploy functions with images in private repositories without having to specify the secret in the `stack.yml` file.

#### Set a custom ImagePullPolicy

Kubernetes allows you to control the conditions for when the Docker images for your functions are pulled onto a node. This is configured through an [imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/#updating-images).

There are three options:

- `Always` - pull the Docker image from the registry every time a deployment changes
- `IfNotPresent` - only pull the image if it does not exist in the local registry cache
- `Never` - never attempt to pull an image

By default, deployed functions will use an `imagePullPolicy` of `Always`, which ensures functions using static image tags (e.g. "latest" tags) are refreshed during an update. This behavior is configurable in `faas-netes` via the `image_pull_policy` environment variable.

If you're using helm you can pass a configuration flag:

```sh
helm upgrade openfaas openfaas/openfaas --install --set "faasnetes.imagePullPolicy=IfNotPresent"
```

If you're using the plain YAML files then edit `gateway-dep.yml` and set the following for `faas-netes`:

```yaml
  - name: image_pull_policy
    value: "IfNotPresent"
```

##### Notes on picking an "imagePullPolicy"

As mentioned above, the default value is `Always`. Every time a function is deployed or is scaled up, Kubernetes will pull a potentially updated copy of the image from the registry. If you are using static image tags like `latest`, this is necessary.

When set to `IfNotPresent`, function deployments may not be updated when using static image tags like `latest`. `IfNotPresent` is particularly useful when developing locally with minikube. In this case, you can set your local environment to use [minikube's docker](https://minikube.sigs.k8s.io/docs/tasks/docker_daemon/) so `faas-cli build` builds directly into the Docker library used by minikube. `faas-cli push` is unnecessary in this workflow - use faas-cli build then faas-cli deploy.

When set to `Never`, only local (or pulled) images will work. This is useful if you want to tightly control which images are available and run in your Kubernetes cluster.
