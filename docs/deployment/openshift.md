# Deployment guide for OpenShift

OpenFaaS is Kubernetes-native and uses *Deployments*, *Services* and *Secrets*. For more detail check out the ["faas-netes" repository](https://github.com/openfaas/faas-netes).

Use this guide to deploy OpenFaaS to an Openshift distribution running an Openshift version between 3.9 and 4.0.

## Build a cluster

You can start evaluating FaaS and building functions on your laptop or on a VM (cloud or on-prem).

* [Install guide for minishift](https://docs.okd.io/latest/minishift/getting-started/installing.html)

* [Deployment information on running Openshift ](https://www.openshift.com/products)

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

### Pick Minishift or Openshift Installation (A or B)

Openshift is slightly different than the Helm installation for the vanilla kubernetes installation, we will need to apply the yaml files recursively. 

#### A. Deploy for Minishift

A minishift addon is avaiable [here](https://github.com/mirroredge/openfaas-minishift-addon)

Once cloned navigate inside the repo run:
``` bash
$ minishift addons install openfaas
```

Then to apply it to the running minishift instance:
``` bash
$ minishift addons apply openfaas
```

To enable it every restart of your minishift instance:
``` bash
$ minishift addons enable openfaas
```

#### B. Deploy for Openshift (for production)

This step assumes you have a running Openshift cluster and have the oc client installed

* Deploy the whole stack

    This command is split into two parts so that the OpenFaaS namespaces are always created first:

    * openfaas - for OpenFaaS services
    * openfaas-fn - for functions

    ```bash
    $ oc new-project openfaas
    $ oc new-project openfaas-fn
    ```

    Now deploy OpenFaaS:

    ```bash
    $ cd faas-netes && \
    oc apply -R -f ./openshift
    oc apply -R -f ./yaml 
    ```

#### Use OpenFaaS

After deploying OpenFaaS you can start using one of the guides or blog posts to create Serverless functions or test [community functions](https://github.com/openfaas/faas/blob/master/community.md).

![](https://camo.githubusercontent.com/72f71cb0b0f6cae1c84f5a40ad57b7a9e389d0b7/68747470733a2f2f7062732e7477696d672e636f6d2f6d656469612f44466b5575483158734141744e4a362e6a70673a6d656469756d)

You can also watch a complete walk-through of OpenFaaS on Kubernetes which demonstrates auto-scaling in action and how to use the Prometheus UI. [Video walk-through](https://www.youtube.com/watch?v=0DbrLsUvaso).

### Deploy a function

The default configuration uses Routes

| Service           | DNS Name                               |
--------------------|----------------------------------------|
| API Gateway / UI  | openfaas-openfaas.<cluster-hostname>   |
| Prometheus        | prometheus-openfaas.<cluster-hostname> |


* Deploy a sample function

There are currently no sample functions built into this stack, but we can deploy them quickly via the UI or FaaS-CLI.

#### Deploy functions from the OpenFaaS Function Store

You can find many different sample functions from the community through the OpenFaaS Function Store. The Function Store is built into the UI portal and also available via the CLI.

You may need to pass the `--gateway` / `-g` flag to each `faas-cli` command or alternatively you can set an environmental variable such as:

```bash
export OPENFAAS_URL=https://openfaas-openfaas.<cluster-hostname>
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

The UI is exposed on https://openfaas-openfaas.<cluster-hostname>.

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
$ echo -n "" | faas-cli invoke --gateway https://openfaas-openfaas.<cluster-hostname> nodeinfo
$ echo -n "verbose" | faas-cli invoke --gateway https://openfaas-openfaas.<cluster-hostname> nodeinfo
```

### Start the hands-on labs

Learn how to build Serverless functions with OpenFaaS and Python in our half-day workshop. You can follow along online at your own pace.

* [OpenFaaS workshop](/tutorials/workshop/)

### Troubleshooting

If you are running into any issues please check out the troubleshooting guide and search the documentation / past issues before raising an issue.

* [Troubleshooting guide](/deployment/troubleshooting/)

### Advanced

This section covers additional advanced topics beyond the initial deployment.


#### Use a private registry with Openshift

If you are using a hosted private Docker registry ([Docker Hub](https://hub.docker.com/), or other),
in order to check how to configure it, please visit the Openshift [documentation](http://v1.uncontained.io/playbooks/continuous_delivery/external-docker-registry-integration.html).

If you try to deploy using `faas-cli deploy` it will fail because the Openshift Deployment component will not have credentials to authorize the docker image pull request.

Once you have pushed an image to a private registry using `faas-cli push` follow the instructions below to either create a pull secret that can be referenced by each function which needs it, or create a secret for the ServiceAccount in the `openfaas-fn` namespace so that any functions which need it can make use of it.

##### Option 1 - use an ad-hoc image pull secret

To deploy your function(s) first you need to create an [Image Pull Secret](https://docs.openshift.com/container-platform/3.11/dev_guide/managing_images.html#using-image-pull-secrets) with the commands below.

Setup some environmental variables:

```bash
export DOCKER_USERNAME=<your_docker_username>
export DOCKER_PASSWORD=<your_docker_password>
export DOCKER_EMAIL=<your_docker_email>
```

Then run this command to create the secret:

```bash
$ oc create secret docker-registry dockerhub \
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
$ oc create secret docker-registry my-private-repo \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-email=$DOCKER_EMAIL \
    --namespace openfaas-fn
```

If needed, pass in the `--docker-server` address.

Use the following command to edit the default ServiceAccount's configuration:

TODO Verify
```sh
$ oc secrets link --for=pull  system:serviceaccount:default -n openfaas-fn
```

Save the changes in the editor and this configuration will be applied.

The OpenFaaS controller will now deploy functions with images in private repositories without having to specify the secret in the `stack.yml` file.

#### Set a custom ImagePullPolicy

Openshift allows you to control the conditions for when the Docker images for your functions are pulled onto a node. This is configured through an [imagePullPolicy](https://docs.openshift.com/container-platform/3.11/dev_guide/managing_images.html#image-pull-policy).

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

As mentioned above, the default value is `Always`. Every time a function is deployed or is scaled up, Openshift will pull a potentially updated copy of the image from the registry. If you are using static image tags like `latest`, this is necessary.

When set to `IfNotPresent`, function deployments may not be updated when using static image tags like `latest`. `IfNotPresent` is particularly useful when developing locally with Minishift.

When set to `Never`, only local (or pulled) images will work. This is useful if you want to tightly control which images are available and run in your Openshift cluster.
