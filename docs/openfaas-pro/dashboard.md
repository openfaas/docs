# OpenFaaS Dashboard

The OpenFaaS Dashboard is a new UI, rebuilt to make operating and understanding OpenFaaS easier.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

## Installation

The OpenFaaS Dashboard is installed through the [openfaas helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas), and will also work with faasd.

Before deploying, you'll need to have created a secret for your license, you'll find instruction in the helm chart for this.

Next, you need to create a JWT signing key and a separate secret for this. It's used to sign and validate logged in sessions.

```bash
# Generate a private key
openssl ecparam -genkey -name prime256v1 -noout -out jwt_key

# Then create a public key from the private key
openssl ec -in jwt_key -pubout -out jwt_key.pub

# Store both in a secret in the openfaas namespace
kubectl -n openfaas \
  create secret generic dashboard-jwt \
  --from-file=key=./jwt_key \
  --from-file=key.pub=./jwt_key.pub
```

To enable the dashboard feature, add the following to your values.yaml file for the openfaas chart:

```yaml
dashboard:
  enabled: true
  publicURL: https://dashboard.example.com
```

The `publicURL`, doesn't necessarily have to be publicly exposed on the Internet, but it does need to be a fully qualified domain name (FQDN).

### Create Ingress to access the dashboard

Once deployed, you will need to create an Ingress record or an Istio VirtualService to access the service.

Assuming you already have a Let's Encrypt Issuer and are using ingress-nginx, you could use the following example:

```bash
export DOMAIN="openfaas.example.com"
export INGRESS_CLASS=nginx

cat > ingress.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: openfaas-dashboard
  namespace: openfaas
  labels:
    app: openfaas-dashboard
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    kubernetes.io/tls-acme: "true"
    $INGRESS_CLASS.ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/ingress.class: $INGRESS_CLASS
spec:
  rules:
  - host: $DOMAIN
    http:
      paths:
      - backend:
          serviceName: openfaas-dashboard
          servicePort: 8080
        path: /
  tls:
  - hosts:
    - $DOMAIN
    secretName: letsencrypt
EOF
```
TLS is mandatory, and you'll use your OpenFaaS password to log in with your browser.

A much simpler alternative for local testing and development is to set up an inlets VM in HTTPS mode: [inlets automated HTTP server](https://docs.inlets.dev/tutorial/automated-http-server/).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inlets-dashboard-client
  namespace: openfaas
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inlets-dashboard-client
  template:
    metadata:
      labels:
        app: inlets-dashboard-client
    spec:
      containers:
      - name: inlets-dashboard-client
        image: ghcr.io/inlets/inlets-pro:0.9.3
        imagePullPolicy: IfNotPresent
        command: ["inlets-pro"]
        args:
        - "http"
        - "client"
        - "--url=wss://EXIT_SERVER_IP"
        - "--token="
        - "--license="
        - "--upstream=dashboard.example.com=http://dashboard.openfaas:8080"
        - "--upstream=gateway.example.com=http://gateway.openfaas:8080"
```

You can use the same tunnel and exit server for multiple domains, to expose both the gateway and the dashboard with TLS and authentication.

## Using the dashboard

Your browser can save the password for your various OpenFaaS environments, so that the credentials can follow you between machines, or be saved in password manager like 1Password.

![Login to the dashboard](/images/dashboard/login-dashboard.png)
> Login to the dashboard

Once logged in, you'll be met by the namespace selector. Namespaces can be used to group functions together, or to provide a level of isolation between teams using the same OpenFaaS installation.

![Select a namespace to explore](/images/dashboard/ns-picker.png)
> Select a namespace to explore

See also: [Namespace support](/reference/namespaces/)

![An overview of functions](/images/dashboard/fn-overview.png)
> An overview of functions

Note that the fields such as repository and SHA can be populated at deploy time and integrate into the UI to create links and show you what's deployed.

![The details for a function](/images/dashboard/details.png)

> View the details for a function, including metrics, logs and metadata about its deployment

![View logs without a terminal, in one place](/images/dashboard/logs.png)

> The logs of the figlet function, viewed without `kubectl` or needing separate terminal access.

```bash
faas-cli store deploy \
    figlet \
    --gateway https://of-pro.example.com \
    --env write_debug=true \
    --env read_debug=true
for i in {0..3}; do curl https://of-pro.example.com/function/figlet -d $i ; done
```

To populate the metadata in the UI, simply set the following at deployment time via `faas-cli deploy` or the OpenFaaS Custom Resource:

| Type | Key | Example value |
|------|-----|---------------|
| annotation | `com.openfaas.git-repo-url` | `https://github.com/openfaas/store-functions` |
| label | `com.openfaas.git-owner` | `openfaas` |
| label | `com.openfaas.git-repo` | `store-functions` |
| label | `com.openfaas.git-branch` | `master` |
| label | `com.openfaas.git-sha` | `665d9597547d8e0425630ba2dbb73c2951a61ce2` |

Here's an example:

```bash
faas-cli store deploy cows \
  --label com.openfaas.scale.min=2 \
  --annotation com.openfaas.git-repo-url=https://github.com/openfaas/store-functions \
  --label com.openfaas.git-owner=openfaas \
  --label com.openfaas.git-repo=store-functions \
  --label com.openfaas.git-branch=master \
  --label com.openfaas.git-sha=f79e2c86e8d67f747d1e449ba6ca63eb5858e5bb
```

Any of these fields can be replaced through environment substitution in stack.yml or using helm during CI/CD.

See also: [Environment substitution in stack.yml](http://localhost:8000/reference/yaml/#yaml-environment-variable-substitution)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
