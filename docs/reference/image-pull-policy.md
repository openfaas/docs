# Using Kuberenetes ImagePullPolicy with OpenFaaS

Kubernetes allows you to control the conditions for when Docker images are pulled onto a node via the [imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/#updating-images) config. Your options are

- `Always` : Kuberenetes will pull the Docker image from the registry every time
- `IfNotPresent` : Kuberentes will only pull the image if it does not exist in the local registry cache
- `Never` : Kuberenetes will never try to pull the image, you must manually ensure that the image already exists in the local cache

By default, deployed functions will use an `imagePullPolicy` of `Always`, which ensures functions using static image tags (e.g. "latest" tags) are refreshed during an update. This behavior is configurable in `faas-netes` via the `image_pull_policy` environment variable. When installing via helm you can easily set this value during install using

```
helm upgrade openfaas openfaas/openfaas --install --set "faasnetesd.imagePullPolicy=IfNotPresent"
```

If installing via a custom yaml manifest, ensure that your `faas-netes` contain spec includes

```
env:
  - name: image_pull_policy
    value: "IfNotPresent"
```

[See here](/deployment/kubernetes/) for more details on deploying OpenFaaS in Kubernetes.

## Which imagePullPolicy should you use

As mentioned above, the default value is `Always`. Every time a function is deployed or is scaled up, Kubernetes will pull a potentially updated copy of the image from the registry. If you are using static image tags like `latest`, this is necessary.

When set to `IfNotPresent`, function deployments may not be updated when using static image tags like `latest`. `IfNotPresent` is particularly useful when developing locally with minikube. In this case, you can set your local environment to use [minikube's docker](https://github.com/kubernetes/minikube/blob/master/docs/reusing_the_docker_daemon.md) so `faas-cli build` builds directly into minikube's image store. `faas-cli push` is unnecessary in this workflow - use faas-cli build then faas-cli deploy.

When set to `Never`, only local (or pulled) images will work. This is useful if you want to tightly control which images are available and run in your Kubernetes cluster.
