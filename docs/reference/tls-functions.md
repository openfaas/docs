## TLS for OpenFaaS Functions

When you expose the [OpenFaaS Gateway via TLS](/reference/tls-openfaas), each function will already be accessible over TLS by adding `/function/NAME` to the URL.

This page is for users who want to create custom domains for individual functions, or to create a REST-like mapping for a number of functions.

It's important that you never expose a function's Pod or Deployment directly, but only via the OpenFaaS gateway. The gateway is required in the invocation path to provide metrics, auto-scaling and asynchronous invocations etc. In order to do this, you will need to create a DNS record and Ingress resource that points to the function's existing path on the gateway service.

In order to automate the process, we built the [ingress-operator](https://github.com/openfaas/ingress-operator) which has its own Custom Resource Definition (CRD) called `FunctionIngress`. The role of `FunctionIngress` is to create an `Ingress` Kubernetes object to map a function to a domain-name, and optionally to also provision a TLS certificate using cert-manager.

Once set up, you'll be able to access a function such as `env` via `https://env.example.com` along with the longer form of: `https://gateway.example.com/function/env`.

### Enable the IngressOperator

You can install the [ingress-operator](https://github.com/openfaas/ingress-operator) by passing `--set ingressOperator.create=true` to the Helm chart at the installation time of OpenFaaS. Or by adding the following to your `values.yaml` file:

```yaml
ingressOperator:
  create: true
```

Run the `helm upgrade --install` command that you used originally to install OpenFaaS.

If you use arkade, add the `--ingress-operator` flag to `arkade install openfaas`.

Check that the operator was deployed and has started:

```bash
$ kubectl get -n openfaas deploy/ingress-operator -o wide

$ kubectl logs -n openfaas deploy/ingress-operator -f
```

### Create a custom domain for a function

#### Deploy a function for testing

Let's deploy a function from the store:

```sh
faas-cli store deploy env
```

If you're using ingress-nginx, then check the public IP with `kubectl get svc/nginxingress-nginx-ingress-controller`, note down the `EXTERNAL-IP`.

Create a DNS A record or CNAME `env.example.com` pointing to the `EXTERNAL-IP`

#### Create a `FunctionIngress` with TLS certificate

FunctionIngress records are always created in the OpenFaaS namespace, so you should have a pre-existing Let's Encrypt Issuer available.

Edit the following fields:

* `tls.enabled` - whether to create the certificate
* `issuerRef.name` - as per the Issuer name created above
* `issuerRef.kind` - optional: either `Issuer` or `ClusterIssuer`

If you're not using ingress-nginx, then also change the `spec.ingressType` field.

The `FunctionIngress` currently makes use of the `HTTP01` challenge, so a separate TLS certificate will be obtained for each FunctionIngress you create.

```sh
export DOMAIN="env.example.com"

cat << EOF > env-fni.yaml
apiVersion: openfaas.com/v1
kind: FunctionIngress
metadata:
  name: env
  namespace: openfaas
spec:
  domain: "env.example.com"
  function: "env"
  ingressType: "nginx"
  tls:
    enabled: true
    issuerRef:
      name: "letsencrypt-prod"
      kind: "Issuer"
EOF

kubectl apply -f env-fni.yaml
```

Verify that the `Certificate` record was created:

```sh
kubectl get cert -n openfaas
```

#### Advanced options for FunctionIngress

* Custom annotations for Ingress

Any annotations that you add to the `FunctionIngress` will be added to the `Ingress` record. This is useful for adding custom annotations that your IngressController supports such as timeouts, rate-limiting or authorization settings.

* FunctionIngress without TLS

The FunctionIngress makes most sense when using TLS, however you can omit TLS for testing by removing the `tls` section of the spec.

* Deleting FunctionIngress records

You can see the `FunctionIngress` records via:

```bash
kubectl get functioningress -n openfaas
```

Then delete one if you need to via: `kubectl delete functioningress/name -n openfaas`.

* Short name for FunctionIngress

The short name is `fni` i.e. `kubectl get fni -n openfaas`.

### Create REST mappings for functions

The FunctionIngress discussed above provides a simple way to create a custom URL mapping scheme for your functions. This is a common request from users, and means that you can map your functions into a REST-style API.

You may wish to map different versions of functions to a top-level domain:

```
https://gateway.example.com/function/env-prod -> https://api.example.com/v1/env/
https://gateway.example.com/function/env-testing -> https://api.example.com/v2/env/
```

You can also use a mapping to perform a blue/green test, by starting off with a mapping rather than the function's name:

```
https://gateway.example.com/function/env-v1 -> https://api.example.com/env/
```

Then, when you're ready, you can deploy a function with a new name and direct traffic there without changing the URL that your users are calling.

```
https://gateway.example.com/function/env-v2 -> https://api.example.com/env/
```

Or you may just prefer your functions to be grouped together under a custom path, rather than at `/function/NAME`.
    
```
https://gateway.example.com/function/create-customer ->   https://api.example.com/v1/customer/
https://gateway.example.com/function/create-order    ->   https://api.example.com/v1/order/
```

#### Map two functions to the same custom domain

This example shows how to map the `env` and `nodeinfo` functions to the same domain under a `v1` path:

```sh
export DOMAIN="api.example.com"

cat << EOF > api-v1-fni.yaml

---
apiVersion: openfaas.com/v1
kind: FunctionIngress
metadata:
  name: env
  namespace: openfaas
spec:
  domain: "$DOMAIN"
  function: "env"
  ingressType: "nginx"
  path: "/v1/env/(.*)"
  tls:
    enabled: true
    issuerRef:
      name: "letsencrypt-prod"
      kind: "Issuer"
---
apiVersion: openfaas.com/v1
kind: FunctionIngress
metadata:
  name: nodeinfo
  namespace: openfaas
spec:
  domain: "$DOMAIN"
  function: "nodeinfo"
  ingressType: "nginx"
  path: "/v1/nodeinfo/(.*)"
  tls:
    enabled: true
    issuerRef:
      name: "letsencrypt-prod"
      kind: "Issuer"
---
```

Apply the FunctionIngress records:

```sh
kubectl apply -f api-v1-fni.yaml
```

You'll now be able to access the above functions via `https://api.example.com/v1/env/` and `https://api.example.com/v1/nodeinfo/`.

