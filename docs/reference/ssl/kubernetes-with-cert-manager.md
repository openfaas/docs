# SSL on Kubernetes

The quickest way to get automated SSL/TLS certificates for your project is to use [cert-manager][cert-manager]. In this tutorial, we will deploy OpenFaaS using the [Helm Chart][openfaas-helm], [cert-manager][cert-manager], and [nginx-ingress][nginx-ingress]

**RBAC** We will assume that your cluster is configured for [RBAC][k8s-rbac].

## Configure Helm and Tiller

First install Helm and the Tiller [following the instructions provided by Helm][helm-install]

## Create namespaces

```sh
$ kubectl create namespace openfaas
$ kubectl create namespace openfaas-fn
```

## Install nginx-ingress

Add the `nginx-ingress` using

```sh
$ helm install stable/nginx-ingress --name nginxingress --set rbac.create=true
```

The full configuration options for nginx [can be found here][nginx-configuration].

## Install cert-manager

We install it into the `kube-system` namespace using the following command:

```sh
$ helm install \
    --name cert-manager \
    --namespace kube-system \
    stable/cert-manager
```

This configuration will work for most deployments, but you can also see https://cert-manager.readthedocs.io/en/latest/getting-started/2-installing.html for additional instructions and options for installing cert-manager.

## Configure cert-manager

In additional to the controller installed in the previous step, we must also configure an "Issuer" before `cert-manager` can create certificates for our services. For convenience we will create an Issuer for both Let's Encrypt's production API and their staging API. The staging API has much kinder rate limits and is useful for testing.

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
    server: https://acme-v01.api.letsencrypt.org/directory
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: openfaas
spec:
  acme:
    server: https://acme-staging.api.letsencrypt.org/directory
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

## Install openfaas

The OpenFaaS Helm Chart already supports the nginx-ingress, but we want to customize it further. This is easiest with a custom values file. Below, we enable and configure the ingress object to use our certificate and expose just the gateway

```yaml
# tls.yml
functionNamespace: openfaas-fn
ingress:
    enabled: true
    annotations:
        kubernetes.io/ingress.class: nginx
        certmanager.k8s.io/cluster-issuer: letsencrypt-prod
    ingress:
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
$ helm repo add openfaas https://openfaas.github.io/faas-netes/
$ helm repo update
$ helm upgrade openfaas \
    --install \
    --namespace openfaas \
    --values tls.yml \
    openfaas/openfaas
```

## Create a certificate

Finally, we can create the Certificate resource which triggers the actual creation of the certificate by `cert-manager`

```yaml
# openfaas-crt.yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: openfaas-crt
spec:
  secretName: openfaas-crt
  dnsNames:
    - openfaas.mydomain.com
  acme:
    config:
      - http01:
          ingressClass: nginx
        domains:
          - openfaas.mydomain.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

```sh
$ kubectl apply -f openfaas-crt.yaml
```

## Deploy and Invoke a function

In your projects containing OpenFaaS functions, you can now update the `provider` defined in the `stack.yaml` to be

```yaml
provider:
  name: faas
  gateway: https://openfaas.mydomain.com
```

## Verify and Debug

There are several commands we can use to verify that the required kubernetes objects have been created.

- To check that the cert-manager issuers were created:
```sh
$ kubectl -n openfaas get issuer letsencrypt-prod letsencrypt-staging
```

- To check that your certificate was created and that cert-manager created the required secret with the actual ssl certificate:
```sh
$ kubectl -n openfaas get certificate,secret openfaas-crt
```

- To check the access and error logs in Nginx
```sh
$ kubectl logs -l "app=nginxingress, component=controller"
```
note that this will print the entire log file, which can be pretty long so we recommend piping the output to `less`
```sh
$ kubectl logs -l "app=nginxingress, component=controller" | less
```


- If you want to tail the Nginx logs, you can use
```sh
$ kubectl logs -f $(kubectl get po -l "app=nginxingress,component=controller" -o jsonpath="{.items[0].metadata.name}")
```

## Profit!

[k8s-rbac]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
[helm]: https://helm.sh
[helm-install]: https://github.com/helm/helm/blob/master/docs/install.md
[nginx-configuration]: https://github.com/helm/charts/tree/master/stable/nginx-ingress#configuration
[openfaas-helm]: https://docs.openfaas.com/deployment/kubernetes/#20a-deploy-with-helm
[cert-manager]: https://github.com/jetstack/cert-manager
[nginx-ingress]: https://github.com/kubernetes/ingress-nginx
