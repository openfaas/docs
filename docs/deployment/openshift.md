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

Once you have deployed OpenFaaS you can create a route to access your gateway and the UI.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: openfaas
  namespace: openfaas
spec:
  tls:
    termination: edge
  to:
    kind: Service
    name: gateway
    weight: 100
  wildcardPolicy: None
```

## Customize your OpenFaaS configuration

The default configuration may not suit all purposes, so if you want to customize your configuration then you can use helm's template command to generate customized YAML which can then be applied with `oc` or `kubectl`.

See also: [template YAML files with helm]](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md#deployment-with-helm-template).

> Note: tiller (the server-side component of helm) *is not required* to generate YAML files

## Test with Minishift

You can also test OpenFaaS with a [minishift add-on](https://github.com/mirroredge/openfaas-minishift-addon) by Mike Schendel
