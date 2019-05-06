# Deployment guide for OpenShift 3.11

OpenFaaS has been tested with OpenShift 3.11.

## Deploy in a container

You can deploy OpenFaaS to OpenShift 3.11 in a container for development and testing.

Tutorial: [Install OpenShift in a container with Weave Footloose](https://blog.alexellis.io/openshift-in-a-footloose-container/) by Alex Ellis

## Deploy to on-premises OpenShift

See the [tutorial above](https://blog.alexellis.io/openshift-in-a-footloose-container/) and start at the section *Test your OpenShift cluster*.

OpenFaaS deploys to two projects (also):

* `openfaas` - core components
* `openfaas-fn` - functions

On a production-grade OpenShift cluster you will need to join the two networks:

```sh
oc adm pod-network join-projects --to=openfaas-fn openfaas
```

## Customize your OpenFaaS configuration

The default configuration may not suit all purposes, so if you want to customize your configuration then you can [use helm to created new YAML files](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md#deployment-with-helm-template) and apply those with `kubectl` or `oc`.

## Test with Minishift

You can also test OpenFaaS with a [minishift add-on](https://github.com/mirroredge/openfaas-minishift-addon) by Mike Schendel
