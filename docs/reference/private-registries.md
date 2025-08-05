## Private registries

You can configure a private container registry to store your OpenFaaS functions.

## Use a private registry with Kubernetes

!!! info "OpenFaaS Standard/for Enterprises"
    Private registry support is part of [OpenFaaS Standard/for Enterprises](/openfaas-pro/introduction).

There are two ways to use private registries with OpenFaaS. This page covers the first use-case.

1. Your functions are private and proprietary, so they are stored in a private container registry. The control-plane components that OpenFaaS offers are available publicly, and do not require a private registry.
2. Your cluster is running in an air-gap or isolated environment. You will need to follow the [airgap instructions](/openfaas-pro/airgap) to set up a private registry for the control-plane components of OpenFaaS.

You can set up your own private Docker registry using this tutorial: [Get a TLS-enabled Docker registry in 5 minutes](https://blog.alexellis.io/get-a-tls-enabled-docker-registry-in-5-minutes/)

Additional steps may be required for cloud hosted registries, if you want to use ambient credentials instead of statically created, or long lived credentials.

AWS Elastic Container Registry (ECR) or Google Container Registry (GCR), Azure Container Registry (ACR), and others in order to authenticate with ambient credentials obtained through the respective cloud provider's metadata endpoints and Identity & Access Management (IAM). This is usually done through a mixture of Pod identity, Kubernetes Service Accounts and a Docker credential helper. For more information, consult the documentation of your cloud registry.

On this page:

- [Option 1 - Link an image pull secret to the namespace's ServiceAccount](#option-1---link-an-image-pull-secret-to-the-namespaces-serviceaccount)
- [Option 2 - Use an ad-hoc image pull secret](#option-2---use-an-ad-hoc-image-pull-secret)
- [Setting an image pull policy](#setting-a-custom-imagepullpolicy)
- [Troubleshooting](#troubleshooting)
- [OpenFaaS Edge & faasd](#openfaas-edge--faasd)

### Option 1 - Link an image pull secret to the namespace's ServiceAccount

Rather than specifying the pull secret for each function that needs it you can bind the secret to the namespace's ServiceAccount. With this option you do not need to update the `secrets:` section of the `stack.yml` file.

Below you will find two examples - one for the Docker Hub, and another that works with any other kind of self-hosted registry.

**Create a Pull Secret for the Docker Hub**

Create the image pull secret in the `openfaas-fn` namespace (or equivalent):

```bash
$ kubectl create secret docker-registry docker-hub-creds \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-email=$DOCKER_EMAIL \
    --namespace openfaas-fn
```

If you're not using the Docker Hub, then pass in the `--docker-server` flag, i.e. `--docker-server=registry.example.com`.

Use the following command to edit the default ServiceAccount's configuration, and to add the imagePullSecret:

```bash
kubectl patch serviceaccount/default \
  -n openfaas-fn default \
  -p '{"imagePullSecrets": [{"name": "docker-hub-creds"}]}'
```

Alternatively, you can edit the list of pull secrets manually:

```sh
$ kubectl edit serviceaccount default -n openfaas-fn
```

At the bottom of the manifest add:

``` yaml
imagePullSecrets:
- name: docker-hub-creds
```

Save the changes in the editor and this configuration will be applied.

The OpenFaaS controller will now deploy functions with images in private repositories without having to specify the secret in the `stack.yml` file.

If you need to deploy from multiple private container registries, repeat the steps above, then include each secret in the list of `imagePullSecrets`.

**A pull secret for a self-hosted registry**

In order to set up a pull secret for a self-hosted registry such as the "OSS distribution", Harbor, jFrog Artifactory, or Quay.io, you can follow the same steps as above, but use the address of your registry in the `--docker-server` flag. The `--docker-email` flag is not required for these registries.

```bash
kubectl create secret docker-registry private-registry \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-server=registry.example.com \
    --namespace openfaas-fn
```

Then patch the ServiceAccount as shown above, i.e.

```bash
kubectl patch serviceaccount/default \
  -n openfaas-fn \
  -p '{"imagePullSecrets": [{"name": "private-registry"}]}'
```

### Option 2 - Use an ad-hoc image pull secret

It is recommended that you patch the service account of the namespace hosting functions, however you can also give the function direct access to a private registry by specifying an image pull secret in the `stack.yml` file.

To deploy your function(s) first you need to create an [Image Pull Secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) with the commands below.

Create a secret as per the steps above. The secret *must* be created in the `openfaas-fn` namespace or the equivalent if it has been changed.

In the `secrets` section of the stack.yaml file, add the name of the secret you created:

```yaml
functions:
  private-fn:
    secrets:
    - dockerhub
```

### Setting a custom ImagePullPolicy

Whilst not required for private registries, it is often worth tuning the default settings for pulling images from a private registry.

Kubernetes allows you to control the conditions for when the Docker images for your functions are pulled onto a node. This is configured through an [imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/#updating-images).

There are three options:

- `Always` - pull the Docker image from the registry every time a deployment changes
- `IfNotPresent` - only pull the image if it does not exist in the local registry cache
- `Never` - never attempt to pull an image

By default, deployed functions will use an `imagePullPolicy` of `Always`, which ensures functions using static image tags (e.g. "latest" tags) are refreshed during an update. 

The value can be configured via Helm in the `values.yaml` file you are maintaining for your installation:

```yaml
functions:
  imagePullPolicy: IfNotPresent
```

To apply the setting, run the `helm upgrade` from the [Getting started guide](/deployment/pro).

## Troubleshooting

If you need to troubleshoot the use of a private image then see the Kubernetes section of the [troubleshooting guide](/deployment/troubleshooting/).

You should also read up on the [Kubernetes docs for registry pull secrets](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

## OpenFaaS Edge & faasd

For OpenFaaS Edge & faasd, configuration see the official handbook: [Serverless For Everyone Else](http://store.openfaas.com/l/serverless-for-everyone-else)
