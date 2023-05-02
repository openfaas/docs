# OpenFaaS Dashboard

The OpenFaaS Dashboard is a new UI, rebuilt to make operating and understanding OpenFaaS easier.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

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
| label | `com.openfaas.git-branch` | `master` |
| label | `com.openfaas.git-owner` | `openfaas` |
| annotation | `com.openfaas.git-repo-url` | `https://github.com/openfaas/store-functions` |
| label | `com.openfaas.git-repo` | `store-functions` |
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

Example snippet from stack.yml:

```yaml
functions:
  cows:
    lang: go
    handler: ./cows
    image: alexellis2/cows:0.1
    labels:
      com.openfaas.git-sha: ${GIT_SHA:-dev}
```

Then using environment variables from your CI system such as GitHub Actions or GitLab CI, override any variables you want to change.

Example of injecting the SHA from Gitlab:

```bash
GIT_SHA=$CI_COMMIT_SHA faas-cli deploy
```

Example with GitHub Actions:

```bash
GIT_SHA=$GITHUB_SHA faas-cli deploy
```

When the GIT_SHA variable is omitted, then the text `dev` will be inputted instead.

See also:

* [GitLab CI Environment variables](https://docs.gitlab.com/ee/ci/variables/#list-all-variables)
* [GitHub Actions Environment variables](https://docs.github.com/en/github-ae@latest/actions/learn-github-actions/variables#default-environment-variables)
* [Environment substitution in stack.yml](/reference/yaml/#yaml-environment-variable-substitution)

## Installation

The OpenFaaS Dashboard is installed through the [openfaas helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas), and will also work with faasd.

Before deploying, you'll need to have created a secret for your license, you'll find instruction in the helm chart for this.

### Create a signing key

You need to create a JWT signing key and a separate secret for the dashboard. It's used to sign and validate logged in sessions.

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

!!! note

    For easy deployment during development this step can be skipped. The dashboard will automatically create the required signing keys on startup.

    Note that these keys will be recreated any time the dashboard container is restarted invalidating any logged in sessions.

### Configure the dashboard

To enable the dashboard feature, add the following to your values.yaml file for the openfaas chart:

```yaml
dashboard:
  enabled: true
  publicURL: https://dashboard.example.com
  # Name of signing key secret for sessions.
  # Can be left blank for development deployments.
  signingKeySecret: "dashboard-jwt" 
```

The `publicURL`, doesn't necessarily have to be publicly exposed on the Internet, but it does need to be a fully qualified domain name (FQDN).

The `signingKeySecret` can be left blank to auto generate signing keys (See: [Create a signing key](#create-a-signing-key)).

### Access your dashboard via port-forwarding

If you don't want to expose your dashboard to users over the Internet, then you can access it as and when required using port-forwarding. Instead of giving a domain and DNS record, you can set the public url in your values.yaml file to `localhost` or an empty string.

```yaml
dashboard:
  enabled: true
  publicURL: "localhost"
```

To access the dashboard run the following, whenever you need it:

```bash
kubectl port-forward \
  -n openfaas \
  svc/dashboard 8081:8080
```

The username is `admin` and the password is the same one used for `faas-cli login`.

Then access the URL via `http://127.0.0.1:8081`

### Expose your dashboard on the Internet

Once deployed, you can leave your dashboard private, and access it via port-forwarding, or create a Public Ingress record to access it over the Internet.

There two three options for Public Ingress:

* Kubernetes Ingress Controller
* Istio Gateway/VirtualService.

If your cluster does not have a public IP, but you still want to access the dashboard over the Internet, we have included notes on how [inlets.dev](https://inlets.dev) can help.

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

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
