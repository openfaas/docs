## TLS for OpenFaaS

Transport Layer Security (TLS) is a cryptographic protocol that provides secure encryption on top of HTTP. It is required for any OpenFaaS gateway which is exposed to the Internet.

This guide explains how to obtain TLS certificates for the OpenFaaS Gateway running on Kubernetes. For faasd, see the instructions in the eBook.

* Setup an Ingress Controller
* Configure cert-manager to obtain a certificate from Let's Encrypt
* Configure the an Ingress record for the OpenFaaS Gateway

### Pre-requisites

* A domain name under your control, and access to create A or CNAME records
* A public IP address with NodePorts, a Load Balancer or a tunnel such as [inlets](https://inlets.dev/)
* A Kubernetes cluster

Where you see `example.com` given in an example, replace that with your own domain name.

### Make sure you can obtain public IP addresses

Managed Kubernetes services have a built-in LoadBalancer provisioner, which will provide a public IP address or CNAME for you, once you create a Service of type LoadBalancer.

If you're running self-managed Kubernetes, where each node has its own Public IP address, then you can configure your Ingress Controller to use a NodePort mapped to port 80 and 443 on the host.

If you are running on a local or private network, you can use [inlets-operator](https://github.com/inlets/inlets-operator) instead, which provisions a VM and uses its public IP address over a websocket tunnel.

### Set up an Ingress Controller

We recommend ingress-nginx for OpenFaaS, however any Ingress controller will work, or you can use Istio with separate instructions.

To install ingress-nginx, use either the Helm chart, or arkade:

```sh
$ arkade install ingress-nginx
```

See also: [ingress-nginx installation](https://kubernetes.github.io/ingress-nginx/deploy/)

### Install cert-manager

cert-manager is a Kubernetes operator maintained by the Cloud Native Computing Foundation (CNCF) which automates TLS certificate management.

To install cert-manager, use either the Helm chart, or arkade:

```sh
$ arkade install cert-manager
```

See also: [cert-manager installation](https://cert-manager.io/docs/installation/)

### Configure cert-manager

You'll need to create an Issuer or ClusterIssuer for your cert-manager installation. This will tell cert-manager which domain it is operating on, and how to register an account for you.

The below will create an Issuer that only operates in the openfaas namespace, with a HTTP01 challenge. Note the ingress class specified in the HTTP01 challenge, this should match the class of your Ingress controller. You can view ingress classes with `kubectl get ingressclass`.

```bash
export EMAIL="you@example.com"

cat > issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: openfaas
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - selector: {}
      http01:
        ingress:
          class: nginx
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: openfaas
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - selector: {}
      http01:
        ingress:
          class: nginx
---

EOF
```

Apply the staging and production Issuers:

```bash
$ kubectl apply -f issuer.yaml
```

### Create the required DNS records

You will need to create an A or CNAME record for your domain, pointing to the public IP address of your Ingress controller.

If you created the Ingress Controller with arkade, you'll see a new service in the default namespace called `ingress-nginx-controller`. You can find the public IP address with:

```sh
$ kubectl get svc -n default ingress-nginx-controller

NAME                       TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.43.87.4   18.136.136.18   80:31876/TCP,443:30108/TCP   28d
```

Take the IP address from the `EXTERNAL-IP` column and create an A record for your domain in your domain management software, or a CNAME record if you're using AWS EKS, and see a domain name in this field.

All users should create an entry for: `gateway.example.com` and then OpenFaaS dashboard users should create an additional record pointing at the same address for: `dashboard.example.com`.

### Configure TLS for the OpenFaaS gateway

You can now configure the OpenFaaS gateway to use TLS by setting the following Helm values, you can save them in a file called `tls.yaml`:

```sh
export DOMAIN="gw.example.com"

cat > tls.yaml <<EOF
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
  tls:
    - hosts:
        - $DOMAIN
      secretName: openfaas-gateway-cert
  hosts:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway
            port:
              number: 8080
EOF
```

If you're using something other than ingress-nginx, then change the `ingressClassName` field accordingly. Note that the `kubernetes.io/ingress.class` annotation is deprecated and should not be used.

The `cert-manager.io/issuer` annotation is used to pick between the staging and production Issuers for Let's Encrypt. If this is your first time working with cert-manager, you may want to use the staging issuer first to avoid running into rate limits if you have something misconfigured.

Now upgrade OpenFaaS via helm, use any custom values.yaml files that you have saved from a previous installation:

```sh
helm repo update && \
    helm upgrade --install openfaas openfaas/openfaas \
        --namespace openfaas \
        --values tls.yaml \
        --values values-custom.yaml
```

### Configure TLS for the OpenFaaS dashboard

If you're using OpenFaaS Standard or OpenFaaS for Enterprises, you will probably want to create an additional Ingress record for the OpenFaaS dashboard.

Edit the previous example:

```sh
export DOMAIN="gw.example.com"
export DOMAIN_DASHBOARD="dashboard.example.com"

cat > tls.yaml <<EOF
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
  tls:
    - hosts:
        - $DOMAIN
      secretName: openfaas-gateway-cert
    - hosts:
        - $DOMAIN_DASHBOARD
      secretName: openfaas-dashboard-cert
  hosts:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway
            port:
              number: 8080
  - host: $DOMAIN_DASHBOARD
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dashboard
            port:
              number: 8080
EOF
```

As above, run the `helm upgrade` command to apply the changes.

### Verifying the installation

First, check that the DNS records you created have taken effect. You can use `nslookup` or `dig` to check that the domain names resolve to the public address of your Ingress Controller's service.

```sh
$ nslookup gw.example.com
```

Next, verify that the Ingress records have the desired domains in the "HOSTS" field:

```sh
$ kubectl get ingress -n openfaas
```

Next, check that the certificates have been issued and that they're ready:

```sh
$ kubectl get certificates -n openfaas -o wide
```

If you're still encountering issues, you can check the logs of the cert-manager controller:

```sh
$ kubectl logs -n cert-manager deploy/cert-manager
```

Log into the OpenFaaS gateway using its new URL:

```sh
$ export OPENFAAS_URL=https://gw.example.com

$ PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
$ echo -n $PASSWORD | faas-cli login --username admin --password-stdin

# List some functions:

$ faas-cli list
```

If you're using Identity and Access Management (IAM) for OpenFaaS, [see the SSO instructions instead](/openfaas-pro/sso/cli.md).
