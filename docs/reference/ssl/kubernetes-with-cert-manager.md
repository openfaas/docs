# TLS on Kubernetes

You can obtain TLS certificates for the OpenFaaS API Gateway and for your functions using [cert-manager][cert-manager] from [JetStack](https://www.jetstack.io).

We will use the following components:

 - [Nginx IngressController][nginx-ingress]
 - OpenFaaS installed via [helm][openfaas-helm]
 - [cert-manager][cert-manager]

We will split this tutorial into two parts:

- 1.0 TLS for the Gateway
- 2.0 TLS and custom domains for your functions
- 3.0 REST-style API mapping for your functions

## 1.0 TLS for the Gateway

This part guides you through setting up all the pre-requisite components to enable TLS for your gateway. You can then access your gateway via a URL such as `https://gw.example.com` and each function such as: `https://gw.example.com/function/nodeinfo`.

### Configure Helm

First install Helm v3 [following the instructions provided by Helm][helm-install]

### Install nginx-ingress

This example will use a Kubernetes [IngressController](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

Add Nginx using the helm `chart-ingress`:

```sh
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm install nginxingress ingress-nginx/ingress-nginx
```

The full configuration options for nginx [can be found here][nginx-configuration].

You will see a service created in the `default` namespace with an `EXTERNAL-IP` of `Pending`, after a few moments it should reveal the public IP allocated by your cloud provider.

```sh
$ kubectl get svc
nginxingress-nginx-ingress-controller        LoadBalancer   192.168.137.172   134.209.179.1
```

Caveats: 

* Alternatively, use [arkade](https://github.com/alexellis/arkade) to install nginx-ingress.

  ```sh
  arkade install ingress-nginx
  ```

* If you do not have a cloud provider for your Kubernetes cluster, but have a public IP, then you can install Nginx in "host-mode" and use the IP of one or more of your nodes for the DNS record.

  ```sh
  $ helm install  nginxingress ingress-nginx/ingress-nginx --set rbac.create=true,controller.hostNetwork=true controller.daemonset.useHostPort=true,dnsPolicy=ClusterFirstWithHostNet,controller.kind=DaemonSet
  ```

  Taken from tutorial: [Setup a private Docker registry with TLS on Kubernetes](https://github.com/alexellis/k8s-tls-registry)

* If you do not have a public IP for your Kubernetes cluster, then you can use a project like [Inlets](https://inlets.dev) and bypass using cert-manager. Inlets has around half a dozen examples of configurations for Kubernetes.

  [HTTPS for your local endpoints with inlets and Caddy](https://blog.alexellis.io/https-inlets-local-endpoints/)

### Install OpenFaaS

Follow the instructions found in the [OpenFaaS Helm Chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#deploy-openfaas). As part of these instructions you will create a basic-auth password to secure the Gateway's API and UI.

Alternatively, use [arkade](https://github.com/alexellis/arkade) to install openfaas:

  ```sh
  arkade install openfaas
  ```

### Create a DNS record

Determine the public IP address which can be used to connect to Nginx:

* If you are using a managed cloud provider, you will receive an IP address in `EXTERNAL-IP`
* If you are using AWS EKS, Nginx will receive a DNS A record in `EXTERNAL-IP`
* If you are using Host Mode for Nginx, then use the IP address of your node

For most people you can create a domain such as `gw.example.com` using a DNS A record, for those using AWS EKS, you will have to create a DNS CNAME entry instead.

> The required steps will vary depending on your domain provider and your cluster provider. For example; [on Google Cloud DNS](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) or [with Route53 using AWS](https://kubernetes.io/docs/setup/custom-cloud/kops/#2-5-create-a-route53-domain-for-your-cluster).

Once created, verify that what you entered into your DNS control-panel worked with `ping`:

```sh
ping gw.example.com
```

You should now see the value you entered. Sometimes DNS can take 1-5 minutes to propagate.

### Install cert-manager

Following the recommended [default installation for cert-manager][cert-manager-helm], we install it into a new `cert-manager` namespace using the following commands:

```sh
# Install the CustomResourceDefinition resources separately
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.11/deploy/manifests/00-crds.yaml

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.11.0 \
  jetstack/cert-manager
```

This configuration will work for most deployments, but you can also see https://cert-manager.readthedocs.io/en/latest/getting-started/install.html#steps for additional instructions and options for installing cert-manager.

### Configure cert-manager

In additional to the controller installed in the previous step, we must also configure an "Issuer" before `cert-manager` can create certificates for our services. For convenience we will create an Issuer for both Let's Encrypt's production API and their staging API. The staging API has much higher rate limits. We will use it to issue a test certificate before switching over to a production certificate if everything works as expected.

Replace `<your-email-here>` with the contact email that will be shown with the TLS certificate.

```yaml
# letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: openfaas
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: <your-email-here>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: openfaas
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: <your-email-here>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```

```sh
$ kubectl apply -f letsencrypt-issuer.yaml
```

This will allow `cert-manager` to automatically provision Certificates just in the `openfaas` namespace.

### Add TLS to openfaas

The OpenFaaS Helm Chart already supports the nginx-ingress, but we want to customize it further. This is easiest with a custom values file. Below, we enable and configure the ingress object to use our certificate and expose just the gateway

```yaml
# tls.yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: "nginx"    
    cert-manager.io/issuer: letsencrypt-staging
  tls:
    - hosts:
        - gw.example.com
      secretName: openfaas-crt
  hosts:
    - host: gw.example.com
      serviceName: gateway
      servicePort: 8080
      path: /
```

```sh
$ helm upgrade openfaas \
    --namespace openfaas \
    --reuse-values \
    --values tls.yaml \
    openfaas/openfaas
```

### Check the certificate

A certificate will be created automatically through "Ingress Shim", part of cert-manager. The Ingress Shim reads annotations to decide which certificates to provision for us.

You can validate that certificate has been obtained successfully using:

```sh
$ kubectl describe certificate \
  -n openfaas \
  openfaas-crt
```

### Switch over to the production issuer

If it was successful, then you can change to the production Let's Encrypt issuer.

* Replace `letsencrypt-staging` with `letsencrypt-prod` in `tls.yaml`
* Run the helm command again

```sh
$ helm upgrade openfaas \
    --namespace openfaas \
    --reuse-values \
    --values tls.yaml \
    openfaas/openfaas
```

### Deploy and Invoke a function

In your projects containing OpenFaaS functions, you can now deploy using your domain as the gateway, replace `gw.example.com` with your domain as well as adding the username and password you created when you deployed OpenFaaS.

```sh
faas-cli login --gateway https://gw.example.com --username <username> --password <password>
faas-cli deploy --gateway https://gw.example.com
```

### Verify and Debug

There are several commands we can use to verify that the required kubernetes objects have been created.

- To check that the cert-manager issuers were created:
```sh
$ kubectl -n openfaas get issuer letsencrypt-prod letsencrypt-staging
```

- To check that your certificate was created and that cert-manager created the required secret with the actual TLS certificate:
```sh
$ kubectl -n openfaas get certificate,secret openfaas-crt
```

- If you want to tail the Nginx logs, you can use
```sh
$ kubectl logs -f $(kubectl get po -l "app=nginxingress,component=controller" -o jsonpath="{.items[0].metadata.name}")
```

## 2.0 TLS and custom domains for functions

This part builds on part 1.0 and now enables custom domains for any of your functions. You will need to have installed OpenFaaS and an IngressController. For TLS, which is optional you need to have cert-manager and at least one `Issuer`.

For example, rather than accessing a function `nodeinfo` via `https://gw.example.com/function/nodeinfo`, you can now use a custom URL such as: `https://nodeinfo.example.com`.

The [IngressOperator](https://github.com/openfaas-incubator/ingress-operator) introduces a new CRD (Custom Resource Definition) called `FunctionIngress`. The role of `FunctionIngress` is to create an `Ingress` Kubernetes object to map a function to a domain-name, and optionally to also provision a TLS certificate using cert-manager.

### Deploy the IngressOperator

You can install the ingress-operator by passing `--set ingressOperator.create=true` to the helm chart at the installation time of OpenFaaS.

[arkade](https://get-arkade.dev/) can also set this flag via `--ingress-operator`

```sh
curl -sSL https://dl.get-arkade.dev | sudo sh

arkade install openfaas --ingress-operator
```

Check that the Operator started correctly:

```
kubectl get deploy/ingress-operator -n openfaas -o wide
```

If it's working, you will see `AVAILABLE` showing `1`. Otherwise use `kubectl logs` or `kubectl get events` for more information.

### Deploy a function

Let's deploy a function from the store:

```sh
faas-cli store deploy nodeinfo
```

Now create a DNS A record in your DNS manager pointing to your IngressController's public IP.

Check the public IP with `kubectl get svc/nginxingress-nginx-ingress-controller`, note down the `EXTERNAL-IP`.

* `nodeinfo.example.com` pointing to the `EXTERNAL-IP`

### Create a `FunctionIngress` Custom Resource (without TLS)

Now create a `FunctionIngress` custom resource:

```yaml
apiVersion: openfaas.com/v1alpha2
kind: FunctionIngress
metadata:
  name: nodeinfo-tls
  namespace: openfaas
spec:
  domain: "nodeinfo-tls.myfaas.club"
  function: "nodeinfo"
  ingressType: "nginx"
```

Verify that the `Ingress` record was created:

```
kubectl get ingress -n openfaas
```

Ingress records are always created in the same namespace as the OpenFaaS Gateway.

### Create a `FunctionIngress` with TLS certificate

To enable TLS, we just need to add the `tls` section and the following fields:

* `tls.enabled` - whether to create the certificate
* `issuerRef.name` - as per the Issuer name created above
* `issuerRef.kind` - optional: either `Issuer` or `ClusterIssuer`

> Note: The `FunctionIngress` currently makes use of the `HTTP01` challenge.

```yaml
apiVersion: openfaas.com/v1alpha2
kind: FunctionIngress
metadata:
  name: nodeinfo-tls
  namespace: openfaas
spec:
  domain: "nodeinfo-tls.myfaas.club"
  function: "nodeinfo"
  ingressType: "nginx"
  tls:
    enabled: true
    issuerRef:
      name: "letsencrypt-staging"
      kind: "Issuer"
```

Verify that the `Certificate` record was created:

```sh
kubectl get cert -n openfaas
```

### Appendix

#### Deleting FunctionIngress records

You can see the `FunctionIngress` records via:

```bash
kubectl get fni -n openfaas
```

Then delete one if you need to via: `kubectl delete fni/name -n openfaas`.

#### Use Zalando's skipper IngressController

[Zalando's skipper IngressController](https://github.com/zalando/skipper) is also supported. To switch over simply add the following to your YAML definition:

```yaml
spec:
  ingressType: "skipper"
```

## 3.0 REST-style API mapping for your functions

The FunctionIngress discussed above provides a simple way to create a custom URL mapping scheme for your functions. This is a common request from users, and means that you can map your functions into a REST-style API.


Here we map three functions to a REST-style API:

```
https://gateway.example.com/function/env -> https://api.example.com/v1/env/
https://gateway.example.com/nodeinfo -> https://api.example.com/v1/nodeinfo/
https://gateway.example.com/certinfo -> https://api.example.com/v1/certinfo/
```

```
faas-cli deploy --image functions/alpine:latest --name env --fprocess env
faas-cli store deploy nodeinfo
faas-cli store deploy certinfo
```

Save the following in a file "fni.yaml" and customise it as required.

Then run `kubectl apply -f fni.yaml`.

To create the TLS Issuer, see the steps above.

```yaml
apiVersion: openfaas.com/v1alpha2
kind: FunctionIngress
metadata:
  name: nodeinfo
  namespace: openfaas
spec:
  domain: "api.example.com"
  function: "nodeinfo"
  ingressType: "nginx"
  path: "/v1/nodeinfo/(.*)"
  tls:
    enabled: true
    issuerRef:
      name: "letsencrypt-staging"
      kind: "Issuer"
---
apiVersion: openfaas.com/v1alpha2
kind: FunctionIngress
metadata:
  name: env
  namespace: openfaas
spec:
  domain: "api.example.com"
  function: "env"
  ingressType: "nginx"
  path: "/v1/env/(.*)"
  tls:
    enabled: true
    issuerRef:
      name: "letsencrypt-staging"
      kind: "Issuer"
---
apiVersion: openfaas.com/v1alpha2
kind: FunctionIngress
metadata:
  name: certinfo
  namespace: openfaas
spec:
  domain: "api.example.com"
  function: "certinfo"
  ingressType: "nginx"
  path: "/v1/certinfo/(.*)"
  tls:
    enabled: true
    issuerRef:
      name: "letsencrypt-staging"
      kind: "Issuer"
```

#### What about IngressController X?

Feel free to raise a feature request for your IngressController on the [GitHub repo](https://github.com/openfaas-incubator/ingress-operator).

[k8s-rbac]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
[helm]: https://helm.sh
[helm-install]: https://github.com/helm/helm/blob/master/docs/install.md
[nginx-configuration]: https://github.com/helm/charts/tree/master/stable/nginx-ingress#configuration
[openfaas-helm]: https://docs.openfaas.com/deployment/kubernetes/#20a-deploy-with-helm
[cert-manager]: https://github.com/jetstack/cert-manager
[cert-manager-helm]: https://cert-manager.readthedocs.io/en/latest/getting-started/install.html#installing-with-helm
[nginx-ingress]: https://github.com/kubernetes/ingress-nginx
