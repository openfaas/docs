# SSL on Kubernetes

You can obtain SSL certificates for the OpenFaaS API Gateway and for your functions using [cert-manager][cert-manager] from [JetStack](https://www.jetstack.io).

We will use the following components:
 - OpenFaaS installed [helm][openfaas-helm] or `helm template`
 - [cert-manager][cert-manager]
 - [Nginx IngressController][nginx-ingress]

We will split this tutorial into two parts:

- 1.0 SSL for the Gateway
- 2.0 SSL and custom domains for your functions

## 1.0 SSL for the Gateway

This part guides you through setting up all the pre-requisite components to enable TLS for your gateway. You can then access your gateway via a URL such as `https://gw.example.com` and each function such as: `https://gw.example.com/function/nodeinfo`.

### Create an A record

If your domain is `.domain.com` then create an A record using your DNS administration panel such as `gateway.domain.com` or `openfaas.domain.com`. The required steps will vary depending on your domain provider and your cluster provider. For example; [on Google Cloud DNS](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) or [with Route53 using AWS](https://kubernetes.io/docs/setup/custom-cloud/kops/#2-5-create-a-route53-domain-for-your-cluster).

### Configure Helm and Tiller

First install Helm and the Tiller [following the instructions provided by Helm][helm-install]

### Install OpenFaaS

Follow the instructions found in the [OpenFaaS Helm Chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#deploy-openfaas), make sure to [secure your gateway with basic auth](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#secure-the-gateway-administrative-api-and-ui-with-basic-auth) before you continue.

### Install nginx-ingress

Add the `nginx-ingress` using

```sh
$ helm install stable/nginx-ingress --name nginxingress --set rbac.create=true
```

The full configuration options for nginx [can be found here][nginx-configuration].

### Install cert-manager

Following the recommended [default installation for cert-manager][cert-manager-helm], we install it into a new `cert-manager` namespace using the following commands:

```sh
# Install the CustomResourceDefinition resources separately
$ kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml

# Create the namespace for cert-manager
$ kubectl create namespace cert-manager

# Label the cert-manager namespace to disable resource validation
$ kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# Add the Jetstack Helm repository
$ helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
$ helm repo update

# Install the cert-manager Helm chart
$ helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.7.0 \
  jetstack/cert-manager
```

This configuration will work for most deployments, but you can also see https://cert-manager.readthedocs.io/en/latest/getting-started/install.html#steps for additional instructions and options for installing cert-manager.

### Configure cert-manager

In additional to the controller installed in the previous step, we must also configure an "Issuer" before `cert-manager` can create certificates for our services. For convenience we will create an Issuer for both Let's Encrypt's production API and their staging API. The staging API has much higher rate limits. We will use it to issue a test certificate before switching over to a production certificate if everything works as expected.

Replace `<your-email-here>` with the contact email that will be shown with the SSL certificate.

```yaml
# letsencrypt-issuer.yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: openfaas
spec:
  acme:
    # Email address used for ACME registration
    email: <your-email-here>
    http01: {}
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      key: ""
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: openfaas
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: <your-email-here>
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    http01: {}
```

```sh
$ kubectl apply -f letsencrypt-issuer.yaml
```

This will allow `cert-manager` to automatically provision Certificates just in the `openfaas` namespace.

### Add TLS to openfaas

The OpenFaaS Helm Chart already supports the nginx-ingress, but we want to customize it further. This is easiest with a custom values file. Below, we enable and configure the ingress object to use our certificate and expose just the gateway

```yaml
# tls.yml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    certmanager.k8s.io/issuer: letsencrypt-staging
    certmanager.k8s.io/acme-challenge-type: http01
  tls:
    - hosts:
        - openfaas.mydomain.com
      secretName: openfaas-crt
  hosts:
    - host: openfaas.mydomain.com
      serviceName: gateway
      servicePort: 8080
      path: /
```


```sh
$ helm upgrade openfaas \
    --namespace openfaas \
    --reuse-values \
    --values tls.yml \
    openfaas/openfaas
```

### Create a certificate

Finally, we can create the Certificate resource which triggers the actual creation of the certificate by `cert-manager`, edit the file below and replace the text `openfaas.mydomain.com` with your address for the API Gateway.

```yaml
# openfaas-crt.yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: openfaas-crt
  namespace: openfaas
spec:
  secretName: openfaas-crt
  issuerRef:
    name: letsencrypt-staging
    kind: Issuer
  dnsNames:
  - openfaas.mydomain.com
  acme:
    config:
    - http01:
        ingressClass: nginx
      domains:
      - openfaas.mydomain.com
```

```sh
$ kubectl apply -f openfaas-crt.yaml
```

You can validate that certificate has been obtained successfully using

```sh
$ kubectl describe certificate openfaas-crt
```

If it was successful you can change to the production Let's Encrypt issuer by replacing `letsencrypt-staging` with `letsencrypt-prod` in the Certificate object in `openfaas-crt.yaml` and re-run `kubectl apply -f openfaas-crt.yaml`.

### Deploy and Invoke a function

In your projects containing OpenFaaS functions, you can now deploy using your domain as the gateway, replace `openfaas.mydomain.com` with your domain as well as adding the username and password you created when you deployed OpenFaaS.

```sh
faas-cli login --gateway https://openfaas.mydomain.com --username <username> --password <password>
faas-cli deploy --gateway https://openfaas.mydomain.com
```

### Verify and Debug

There are several commands we can use to verify that the required kubernetes objects have been created.

- To check that the cert-manager issuers were created:
```sh
$ kubectl -n openfaas get issuer letsencrypt-prod letsencrypt-staging
```

- To check that your certificate was created and that cert-manager created the required secret with the actual ssl certificate:
```sh
$ kubectl -n openfaas get certificate,secret openfaas-crt
```

- If you want to tail the Nginx logs, you can use
```sh
$ kubectl logs -f $(kubectl get po -l "app=nginxingress,component=controller" -o jsonpath="{.items[0].metadata.name}")
```

## 2.0 SSL and custom domains for functions

This part builds on part 1.0 and now enables custom domains for any of your functions. You will need to have installed OpenFaaS and an IngressController. For TLS, which is optional you need to have cert-manager and at least one `Issuer`.

For example, rather than accessing a function `nodeinfo` via `https://gw.example.com/function/nodeinfo`, you can now use a custom URL such as: `https://nodeinfo.example.com`.

The [IngressOperator](https://github.com/openfaas-incubator/ingress-operator) introduces a new CRD (Custom Resource Definition) called `FunctionIngress`. The role of `FunctionIngress` is to create an `Ingress` Kubernetes object to map a function to a domain-name, and optionally to also provision a TLS certificate using cert-manager.

### Deploy the IngressOperator

```sh
git clone https://github.com/openfaas-incubator/ingress-operator
cd ingress-operator

kubectl apply -f ./artifacts/operator-crd.yaml
kubectl apply -f ./artifacts/operator-rbac.yaml
kubectl apply -f ./artifacts/operator-amd64.yaml
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

### Create a `FunctionIngress` Custom Resource (without SSL)

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

### Create a `FunctionIngress` with SSL certificate

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

### Use Zalando's skipper IngressController

[Zalando's skipper IngressController](https://github.com/zalando/skipper) is also supported. To switch over simply add the following to your YAML definition:

```yaml
spec:
  ingressType: "skipper"
```

### What about IngressController X?

Feel free to raise a feature request for your IngressController on the [GitHub repo](https://github.com/openfaas-incubator/ingress-operator).


[k8s-rbac]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
[helm]: https://helm.sh
[helm-install]: https://github.com/helm/helm/blob/master/docs/install.md
[nginx-configuration]: https://github.com/helm/charts/tree/master/stable/nginx-ingress#configuration
[openfaas-helm]: https://docs.openfaas.com/deployment/kubernetes/#20a-deploy-with-helm
[cert-manager]: https://github.com/jetstack/cert-manager
[cert-manager-helm]: https://cert-manager.readthedocs.io/en/latest/getting-started/install.html#installing-with-helm
[nginx-ingress]: https://github.com/kubernetes/ingress-nginx
