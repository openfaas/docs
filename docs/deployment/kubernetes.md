# Deployment guide for Kubernetes

OpenFaaS is Kubernetes-native and uses *Deployments*, *Services* and *Secrets*. For more detail check out the ["faas-netes" repository](https://github.com/openfaas/faas-netes).

Use this guide to deploy OpenFaaS to a vanilla Kubernetes distribution running a Kubernetes version between 1.8 and 1.13.

## Build a cluster

Before deploying OpenFaaS, you should provision a Kubernetes cluster. There are many options for deploying a local or remote cluster. You can read about the [various Kubernetes distributions here](https://kubernetes.io/docs/setup/).

Once you have a cluster, you can follow the detailed instructions on this page.

* Install OpenFaaS CLI
* Deploy OpenFaaS from static YAML, via helm, or via new YAML files generated with `helm template`
* Find your OpenFaaS gateway address
* Log in, deploy a function, and try out the UI.

> If you should need technical support, then see the [Community Page](/community/).

### Local clusters

* [k3s](https://k3s.io) - a light-weight Kubernetes distribution ideal for edge and development - compatible with Raspberry Pi & ARM64 (Packet, AWS Graviton)
* [k3d](https://github.com/rancher/k3d) - makes k3s available on any computer where Docker is also running
* [minikube](https://minikube.sigs.k8s.io) - a popular, but heavy-weight option that creates a Linux virtual machine your computer using VirtualBox or similar
* [Docker for Mac/Windows](https://docs.docker.com/docker-for-mac/install/) - Docker's Desktop edition has an option to run a local Kubernetes cluster

### Remote/managed options

You can run `k3s` and `k3d` on a single node Virtual Machine so that you don't have to run Kubernetes on your own computer.

* The [k3sup (ketchup)](https://k3sup.dev) tool can help you to do this by installing k3s onto a remote VM

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

You can install the OpenFaaS CLI using `brew` or a `curl` script.

* via `brew`:

```bash
brew install faas-cli
```

* via `curl`:

```bash
$ curl -sL https://cli.openfaas.com | sudo sh
```

If you run the script as a normal non-root user then the script will be downloaded to the current folder.

### Pick helm or YAML files for deployment (A or B)

It is recommended to use `helm` to install OpenFaaS so that you can configure your installation to suit your needs. This configuration is considered to be production-ready.

Plain YAML files are also provided for x86_64 and armhf, but since they cannot be customized easily it is recommended that you only use these for local development.

#### A. Deploy with Helm (for production)

A Helm chart is provided in the `faas-netes` repository. Follow the link below then come back to this page.

* [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/blob/master/HELM.md)

    !!! note
        Some users may have concerns about using helm charts due to security concerns with the `tiller` component. If you fall into this category of users, then don't worry, you can still benefit from the helm chart without using `tiller`.
        
        See the [Chart readme](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md#deployment-with-helm-template) for how to generate your own static YAML files using `helm template`.

#### B. Deploy using kubectl/YAML (for development-only)

This step assumes you are running `kubectl` on a master host.

* Clone the code

    ```bash
    $ git clone https://github.com/openfaas/faas-netes
    ```

    Deploy a stack with asynchronous functionality provided by NATS Streaming.

* Deploy the whole stack

    This command is split into two parts so that the OpenFaaS namespaces are always created first:

    * openfaas - for OpenFaaS services
    * openfaas-fn - for functions

    ```bash
    $ kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
    ```

    Create a password for the gateway:

    ```bash
    # generate a random password
    PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)

    kubectl -n openfaas create secret generic basic-auth \
    --from-literal=basic-auth-user=admin \
    --from-literal=basic-auth-password="$PASSWORD"
    ```

    Now deploy OpenFaaS:

    ```bash
    $ cd faas-netes && \
    kubectl apply -f ./yaml
    ```

    Set your `OPENFAAS_URL`, if using a NodePort this may be `127.0.0.1:31112`.

    If you're using a remote cluster, or you're not sure then you can also port-forward the gateway to your machine for this step.

    ```bash
    kubectl port-forward svc/gateway -n openfaas 31112:8080 &
    ```

    Now log in:
    ```bash
    export OPENFAAS_URL=http://127.0.0.1:31112

    echo -n $PASSWORD | faas-cli login --password-stdin
    ```

    !!! note
        For deploying on a cloud that supports Kubernetes *LoadBalancers* you may also want to apply the configuration in: `cloud/lb.yml`.

#### Raspberry Pi & 32-bit ARM (armhf)

> For a complete tutorial on setting up OpenFaaS for Raspberry Pi / 32-bit ARM using Kubernetes see the following blog post from Alex Ellis: [Will it Cluster?](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/).

For Raspberry Pi or 32-bit ARM devices please do the following:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```

Clone the GitHub repository:

```bash
$ git clone https://github.com/openfaas/faas-netes
```

Now deploy OpenFaaS:

```bash
$ cd faas-netes && \
kubectl apply -f ./yaml_armhf
```

When creating new functions please use the templates with a suffix of `-armhf` such as `go-armhf` and `python-armhf` to ensure you get the correct versions for your devices.

> Note: you cannot deploy the sample functions to ARM devices, but you can use the function store in the gateway UI or via `faas-cli store list --yaml https://raw.githubusercontent.com/openfaas/store/master/store-armhf.json`

#### 64-bit ARM and AWS Graviton

For 64-bit ARM servers and devices such as ODroid-C2, Rock64, AWS Graviton and the servers provided by [Packet.net](https://packet.net/) please run the following:


* Create the project namespaces

```bash
$ kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```
Clone the GitHub repository:

```bash
$ git clone https://github.com/openfaas/faas-netes
```

Now deploy OpenFaaS:

```bash
$ cd faas-netes && \
kubectl apply -f ./yaml_arm64
```

* Create a password

```bash
# generate a random password
PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)

kubectl -n openfaas create secret generic basic-auth \
--from-literal=basic-auth-user=admin \
--from-literal=basic-auth-password="$PASSWORD"
```

Set your `OPENFAAS_URL`, if using a NodePort this may be `127.0.0.1:31112`.

If you're using a remote cluster, or you're not sure then you can also port-forward the gateway to your machine for this step.

```bash
kubectl port-forward svc/gateway -n openfaas 31112:8080 &
```

Now log in:
```bash
export OPENFAAS_URL=http://127.0.0.1:31112

echo -n $PASSWORD | faas-cli login --password-stdin
```

When creating new functions please use the templates with a suffix of `-arm64` such as `node-arm64` to ensure you get the correct versions for your devices.

> Note: you cannot deploy the sample functions to ARM64 devices, but you can use the function store in the gateway UI or via `faas-cli store list --yaml https://raw.githubusercontent.com/openfaas/store/master/store-arm64.json`

#### Use OpenFaaS

The community has built a workshop with 12 self-paced hands-on labs. Use the workshop to begin learning OpenFaaS at your own pace:

* [OpenFaaS workshop](/tutorials/workshop/)

You can also find a list of [community tutorials, events, and videos](https://github.com/openfaas/faas/blob/master/community.md).

![](https://camo.githubusercontent.com/72f71cb0b0f6cae1c84f5a40ad57b7a9e389d0b7/68747470733a2f2f7062732e7477696d672e636f6d2f6d656469612f44466b5575483158734141744e4a362e6a70673a6d656469756d)

A walk-through video shows auto-scaling in action and the Prometheus UI: [walk-through video](https://www.youtube.com/watch?v=0DbrLsUvaso).

### Deploy a function

For ease of use, the default configuration uses NodePorts rather than an IngressController or LoadBalancer.

| Service           | TCP port |
--------------------|----------|
| API Gateway / UI  | 31112    |
| Prometheus        | 31119    |

!!! note
    Advanced users can configure Ingress or a LoadBalancer in the helm chart.

Functions can be deployed using the REST API, UI, CLI, or Function Store. Continue below to deploy your first sample function.

#### Deploy functions from the OpenFaaS Function Store

You can find many different sample functions from the community through the OpenFaaS Function Store. The Function Store is built into the UI portal and also available via the CLI.

You may need to pass the `--gateway` / `-g` flag to each `faas-cli` command or alternatively you can set an environmental variable such as:

```bash
export OPENFAAS_URL=http://127.0.0.1:31112
```

To search the store:

```bash
$ faas-cli store list
```

To deploy `figlet`:

```bash
$ faas-cli store deploy figlet
```

Now find the function deployed in the cluster and invoke it.

```bash
$ faas-cli list
$ echo "OpenFaaS!" | faas-cli invoke figlet
```

You can also access the Function Store from the Portal UI and find a range of functions covering everything from machine-learning to network tools.

##### Build your first Python function

[Your first serverless Python function with OpenFaaS](https://blog.alexellis.io/first-faas-python-function/)

#### Use the UI

The UI is exposed on NodePort 31112.

Click "New Function" and fill it out with the following:

| Field      | Value                        |
-------------|------------------------------|
| Service    | nodeinfo                     |
| Image      | functions/nodeinfo:latest    |
| fProcess   | node main.js                 |
| Network    | default                      |

* Test the function

Your function will appear after a few seconds and you can click "Invoke"

The function can also be invoked through the CLI:

```bash
$ echo -n "" | faas-cli invoke --gateway http://kubernetes-ip:31112 nodeinfo
$ echo -n "verbose" | faas-cli invoke --gateway http://kubernetes-ip:31112 nodeinfo
```

### Start the hands-on labs

Learn how to build Serverless functions with OpenFaaS and Python in our half-day workshop. You can follow along online at your own pace.

* [OpenFaaS workshop](/tutorials/workshop/)

### Troubleshooting

If you are running into any issues please check out the troubleshooting guide and search the documentation / past issues before raising an issue.

* [Troubleshooting guide](/deployment/troubleshooting/)

### Advanced

This section covers additional advanced topics beyond the initial deployment.

#### Deploy with SSL

To enable SSL while using Helm, try one of the following references:

* [Using nginx-ingress and cert-manager](/reference/ssl/kubernetes-with-cert-manager/)

#### Use a private registry with Kubernetes

If you are using a hosted private Docker registry ([Docker Hub](https://hub.docker.com/), or other),
in order to check how to configure it, please visit the Kubernetes [documentation](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry).

If you try to deploy using `faas-cli deploy` it will fail because the Kubernetes kubelet component will not have credentials to authorize the docker image pull request.

Once you have pushed an image to a private registry using `faas-cli push` follow the instructions below to either create a pull secret that can be referenced by each function which needs it, or create a secret for the ServiceAccount in the `openfaas-fn` namespace so that any functions which need it can make use of it.

If you need to troubleshoot the use of a private image then see the Kubernetes section of the [troubleshooting guide](./troubleshooting.md).

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
