## Function Custom Resource Definition (CRD)

In addition to the REST API of the OpenFaaS gateway, the `Function` Custom Resource Definition can be used to manage functions.

Glossary:

* CRD - the definition of a Function, the Custom Resource Definition (CRD) is installed to the cluster via Helm, once.
* Each instance of a Function is called a Custom Resource (CR).

!!! warning "Do not remove the CRD"

    Do not remove the Custom Resource Definition from the cluster.

    If you remove the Function CRD, then all instances of the CRD (known as CRs) will be removed from the cluster, meaning all your functions will be deleted immediately.

See also:

* [How and why you should upgrade to the Function Custom Resource Definition (CRD)](https://www.openfaas.com/blog/upgrade-to-the-function-crd/)
* [How to package OpenFaaS functions with Helm](https://www.openfaas.com/blog/howto-package-functions-with-helm/)

### Function Spec

Here is an example of a function, written out in long form:

```yaml
---
apiVersion: openfaas.com/v1
kind: Function
metadata:
  name: nodeinfo
  namespace: openfaas-fn
spec:
  name: nodeinfo
  handler: node main.js
  image: functions/nodeinfo:latest
  labels:
    com.openfaas.scale.min: "2"
    com.openfaas.scale.max: "15"
  annotations:
    current-time: Mon 6 Aug 23:42:00 BST 2018
    next-time: Mon 6 Aug 23:42:00 BST 2019
  environment:
    write_debug: "true"
  limits:
    cpu: "200m"
    memory: "256Mi"
  requests:
    cpu: "10m"
    memory: "128Mi"
  constraints:
    - "cloud.google.com/gke-nodepool=default-pool"
  secrets:
    - nodeinfo-secret1
```

## How to generate a spec from a stack.yml

If you already have functions defined in a stack.yml file, you can generate the Function resources with:

```bash
faas-cli generate [-f stack.yml]
```

If you'd like to generate a Function CR from a store-function, run:

```bash
faas-cli generate --from-store env
```

Either command can be piped into a file, or directly into `kubectl apply`

```bash
faas-cli generate > functions.yaml

faas-cli generate --from-store env > env.yaml

faas-cli generate --from-store nodeinfo | kubectl apply -f-
```

### Environment substitution

Whilst generating the Function Custom Resource from a stack.yml file, you can also substitute text such as within labels or the image tag.

stack.yml

```yaml
provider:
  name: openfaas

functions:
  bcrypt:
    lang: golang-middleware
    handler: ./bcrypt
    image: ghcr.io/${OWNER:-alexellis}/bcrypt:${TAG:-latest}
    labels:
      com.openfaas.scale.min: 1
      com.openfaas.scale.max: 10
      com.openfaas.scale.target: 500
      com.openfaas.scale.type: cpu

      com.openfaas.git-branch: master
      com.openfaas.git-owner: ${OWNER:-alexellis}
      com.openfaas.git-repo: autoscaling-functions
      com.openfaas.git-sha: ${TAG:-4089a5f18b0692833e28a88d9ca263a68c329034}

    annotations:
      com.openfaas.git-repo-url: https://github.com/${OWNER:-alexellis}/autoscaling-functions
      github: true
```

In the example above, there is a default owner for the: image owner, image tag, owner and tag in two labels, and in an annotation.

Therefore, in CI, you may run something like:

```bash
TAG=$CI_TAG OWNER=$CI_OWNER faas-cli generate | kubectl apply -f -
```

### Readiness and health

The OpenFaaS REST API contains readiness information in the form of available replicas being > 0. The `faas-cli` also has a `describe` command which queries the amount of available replicas and will report Ready or Not Ready.

That said, the Custom Resource now has a `.status` field with a number of conditions.

* `Reconciling` is `True` - the operator has discovered the Function CR and is working on creating child resources in Kubernetes
* `Stalled` is `True` - the operator cannot create the function, check the reason, perhaps some secrets are missing?
* `Ready` condition is `True` - the operator was able to create the required Deployment, all required secrets exist, and it has no more work to do. The `Reconciling` condition is removed
* `Healthy` condition is `True` - there is at least one ready Pod available to serve traffic

What about scaling to zero? When a Function is scaled to zero, the Ready state will be `True` and the Healthy state will be `False`. Invoke the function via its URL on the gateway to have it scale up from zero, you'll see it progress and gain a `Healthy` condition of `True`.

If you change a Function once it has passed into the Ready=true state, then the Ready condition will change to Unknown, and the Reconciling condition will be added again.

If you're an ArgoCD user, then you can define a custom health check at deployment time to integrate with the Custom Resource: [ArgoCD: Custom Health Checks](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#custom-health-checks)

To check that all secrets existed, and that the function could be applied correctly, then look for a `Ready` condition with a status of `True` for the current generation of the object. If you want to know if the function's code was valid, and was able to start, and that its image could be pulled, look for the `Healthy` condition instead. Bear in mind that any functions scaled to zero may have the Healthy condition removed, or set to False.

What happens when auto-scaling? When a Function is scaling from at least one replica to some higher number, the Healthy condition will continue to be set to `True`. The same applies when scaling down, so long as the target number of replicas is greater than zero.

### Deploy functions via Helm

You can template and deploy functions via Helm, ArgoCD and Flux by including the Function's CR YAML in the same way you would for any other native Kubernetes resource.

See also: [How to package OpenFaaS functions with Helm](https://www.openfaas.com/blog/howto-package-functions-with-helm/)
