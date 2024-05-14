# Use a local Ingress Controller with KinD

Most users will use port-forwarding to access the OpenFaaS gateway, it's the simplest option and works everywhere.

However, in this tutorial, we will show you how to deploy OpenFaaS with ingress-nginx.

When you use an Ingress Controller:

* You can access the gateway on your host machine without port-forwarding
* If you update the gateway, you don't need to restart port-forwarding commands
* OpenFaaS is almost always exposed behind an Ingress Controller in production, so it's good practice to use it locally

Is this tutorial for you?

* If you want to access the local OpenFaaS gateway over the Internet with a TLS certificate, see: [Expose your local OpenFaaS functions to the Internet](https://inlets.dev/blog/2020/10/15/openfaas-public-endpoints.html)
* If you are deploying to a cluster on the public Internet, see: [TLS on Kubernetes](/reference/ssl/kubernetes-with-cert-manager/)

## Prerequisites

You'll need Docker running, plus:

* `kind`
* `kubectl`
* `faas-cli`

You can get the above with [arkade](https://arkade.dev).

Run: `arkade get kind kubectl faas-cli`.

## Create a KinD cluster with ports 80 and 443 exposed

```sh
cat > kind-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

kind create cluster --name openfaas --config kind-config.yaml
```

## Install the ingress-nginx IngressController

Use arkade, or [install ingress-nginx manually](https://kubernetes.github.io/ingress-nginx/deploy/).

```sh
arkade install ingress-nginx
```

## Install OpenFaaS with local Ingress enabled

Usually, Ingress is used when a cluster has a public IP address, and you want to obtain TLS certificates from Let's Encrypt. In this case, we'll use it to access the OpenFaaS gateway on the host machine.

Create values-ingress.yaml:

```yaml
ingress:
  exposeServices: false
  enabled: true

  hosts:
    - host: openfaas.local
      serviceName: gateway
      servicePort: 8080
      path: /
  ingressClassName: nginx
  annotations:
    annotations.kubernetes.io/ingress.class: nginx
```

On Linux, MacOS or WSL2, edit your `/etc/hosts` file and add an entry:

```
127.0.0.1    openfaas.local
```

Now install OpenFaaS via the Helm chart in the usual way, but make sure you also pass `-f values-ingress.yaml`, along with your custom values.yaml file.

## Access OpenFaaS via Ingress

You can now access OpenFaaS via the URL of `http://openfaas.local`.

```sh
export OPENFAAS_URL=http://openfaas.local

faas-cli login 

faas-cli store deploy env

faas-cli list
```

