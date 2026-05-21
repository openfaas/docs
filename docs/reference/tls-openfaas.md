# TLS for OpenFaaS

Transport Layer Security (TLS) is a cryptographic protocol that provides secure encryption on top of HTTP. It is required for any OpenFaaS gateway which is exposed to the Internet.

This guide explains how to obtain TLS certificates for the OpenFaaS Gateway running on Kubernetes. For faasd, see the instructions in the eBook.

## Choosing an approach

There are two ways to route external HTTPS traffic to the OpenFaaS gateway on Kubernetes:

* **[Gateway API](#gateway-api)** — The recommended approach for new installations. Uses Kubernetes Gateway API resources for traffic routing and TLS.
    * [Envoy Gateway](#gateway-api-with-envoy-gateway) — General setup using Envoy Gateway and cert-manager.
    * [AWS EKS](#aws-eks-with-gateway-api) — AWS specific options using the AWS Load Balancer Controller.
* **[Ingress](#ingress)** — A good option for simple setups, existing installations, or clusters that cannot yet migrate to the Gateway API.
    * [Ingress controller](#setup-an-ingress-controller) — General setup with Traefik as the Ingress Controller and cert-manager for Let's Encrypt certificates.
    * [AWS EKS](#aws-eks-install-traefik-with-nlb) — Additional AWS specific steps to provision an Network Load Balancer (NLB) using the AWS Load Balancer Controller.

## Pre-requisites

* A domain name under your control, and access to create A or CNAME records
* A public IP address with NodePorts, a Load Balancer or a tunnel such as [inlets](https://inlets.dev/)
* A Kubernetes cluster with OpenFaaS installed via Helm

Where you see `example.com` given in an example, replace that with your own domain name.

### Make sure you can obtain public IP addresses

Managed Kubernetes services have a built-in LoadBalancer provisioner, which will provide a public IP address or CNAME for you, once you create a Service of type LoadBalancer.

If you're running self-managed Kubernetes, where each node has its own Public IP address, then you can configure your Ingress Controller or Gateway proxy to use a NodePort mapped to port 80 and 443 on the host.

If you are running on a local or private network, you can use [inlets-operator](https://github.com/inlets/inlets-operator) instead, which provisions a VM and uses its public IP address over a websocket tunnel.

## Gateway API

This section covers setting up TLS using [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) resources. The general instructions use Envoy Gateway with cert-manager. If you are running on AWS EKS, see the [AWS-specific options](#aws-eks-with-gateway-api) after reviewing the general setup.

From the perspective of OpenFaaS, there are three Gateway API resources we need:

* **GatewayClass** - maps to IngressClass - i.e. whether you're using Envoy Gateway, Traefik, NGINX Gateway Fabric, Istio, or another implementation.
* **Gateway** - maps to a LoadBalancer Service with one or more listeners and handles TLS configuration.
* **HTTPRoute** - binds paths and/or hostnames to backend services.

```
    Internet (HTTPS)
           │
           ▼
     Public IP ─────> DNS: gw.example.com
           │
           ▼
  ┌─────────────────┐
  │    Gateway      │  GatewayClass (e.g. Envoy Gateway)
  │                 │  :80, :443 (TLS terminates here)
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │   HTTPRoute     │  Host: gw.example.com, Path: /
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ gateway Service │  :8080
  └────────┬────────┘
           │
           ▼
    OpenFaaS Gateway
```

### Gateway API with Envoy Gateway

This guide uses [Envoy Gateway](https://gateway.envoyproxy.io/) as the Gateway API implementation. If you want to use a different [conformant implementation](https://gateway-api.sigs.k8s.io/implementations/), install it using its own documentation and change the `gatewayClassName` in the rest of the guide. 

#### Check and update Gateway API CRDs

Some Kubernetes distributions ship their own version of the Gateway API CRDs, which may not match those your implementation wants to use.

Check with:

```sh
kubectl get crd | grep gateway.networking.k8s.io
```

For this example, it's best to let Envoy Gateway handle the CRD installation with versions it supports, so remove any pre-existing CRDs:

```sh
# Replace v1.1.0 with the version installed in your cluster
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/experimental-install.yaml
```

#### Install Envoy Gateway

Install Envoy Gateway using [its Helm chart](https://gateway.envoyproxy.io/docs/install/install-helm/). The chart includes the Gateway API CRDs, so no separate CRD installation is needed.

Bear in mind that Envoy Gateway maintains its own [compatibility matrix](https://gateway.envoyproxy.io/news/releases/matrix/).

```sh
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.0 \
  -n envoy-gateway-system \
  --create-namespace
```

Wait for Envoy Gateway to become available:

```sh
kubectl wait --timeout=5m -n envoy-gateway-system \
  deployment/envoy-gateway --for=condition=Available
```

#### Create a GatewayClass

Create a GatewayClass for Envoy Gateway:

```bash
cat > gatewayclass.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
EOF
```

```bash
kubectl apply -f gatewayclass.yaml
```

#### Install cert-manager

[cert-manager](https://cert-manager.io) automates TLS certificate management in Kubernetes. It integrates with the Gateway API to automatically create certificates for Gateway listeners.

Install cert-manager with Gateway API support enabled:

```sh
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true
```

The `enableGatewayAPI` flag tells cert-manager to watch for Gateway API resources when solving ACME challenges.

!!! note
    The Gateway API CRDs must be installed before cert-manager starts. If you installed them after cert-manager, restart the controller with: `kubectl rollout restart deployment cert-manager -n cert-manager`

See also: [cert-manager installation](https://cert-manager.io/docs/installation/)

#### Create a cert-manager Issuer

Create an Issuer in the `openfaas` namespace that uses Let's Encrypt with an HTTP-01 challenge.

```bash
cat > issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: openfaas
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: openfaas-gateway
                namespace: openfaas
                kind: Gateway
EOF
```

```bash
kubectl apply -f issuer.yaml
```

The `parentRefs` field points to the Gateway we'll create in the next step, so cert-manager knows which Gateway to attach the challenge route to. The referenced Gateway must have a listener on port 80, since the HTTP-01 challenge requires Let's Encrypt to reach a well-known URL over plain HTTP.

#### Create the Gateway

Create a Gateway with an HTTP listener (for ACME challenges) and an HTTPS listener for the OpenFaaS gateway:

```bash
export GW_DOMAIN="gw.example.com"

cat > gateway.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openfaas-gateway
  namespace: openfaas
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
    - name: gateway
      hostname: "$GW_DOMAIN"
      port: 443
      protocol: HTTPS
      allowedRoutes:
        namespaces:
          from: Same
      tls:
        mode: Terminate
        certificateRefs:
          - name: openfaas-gateway-cert
EOF
```

```bash
kubectl apply -f gateway.yaml
```

The `cert-manager.io/issuer` annotation tells cert-manager to watch this Gateway and automatically create a Certificate resource for each HTTPS listener. The certificate will be stored in the Secret referenced by `certificateRefs`.

The first listener on port 80 is for HTTP-01 challenges. The second listener serves HTTPS traffic for `gw.example.com` on port 443. The `tls.mode: Terminate` setting means TLS is terminated at the Gateway and traffic is forwarded to the backend as plain HTTP.

#### Create the DNS record

Find the external IP address assigned to the Gateway:

```sh
kubectl get gateway -n openfaas openfaas-gateway

NAME               CLASS   ADDRESS          PROGRAMMED
openfaas-gateway   eg      203.0.113.10     True
```

Create an A record (or CNAME if you see a hostname) in your DNS provider pointing `gw.example.com` to this address.

#### Create the HTTPRoute

While the Gateway defines listeners and TLS termination, it is the HTTPRoute that binds hostnames and paths to backend services.

Create an HTTPRoute that routes traffic from the Gateway to the OpenFaaS gateway service:

```bash
export GW_DOMAIN="gw.example.com"

cat > httproute.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openfaas-gateway
  namespace: openfaas
spec:
  parentRefs:
    - name: openfaas-gateway
  hostnames:
    - "$GW_DOMAIN"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      timeouts:
        request: 10m
      backendRefs:
        - name: gateway
          port: 8080
EOF
```

```bash
kubectl apply -f httproute.yaml
```

The `parentRefs` field defines which Gateway this route is attached to. The `hostnames` field filters requests by the Host header, ensuring only requests for `gw.example.com` are matched. The `backendRefs` field forwards matching requests to the OpenFaaS `gateway` service on port 8080.

The `timeouts.request` field sets the maximum duration for the gateway to respond to an HTTP request. This value should be set to match the `gateway.writeTimeout` configured in the OpenFaaS Helm chart. If omitted, Envoy Proxy uses a default of 15 seconds which will cause functions with longer execution times to time out at the proxy level. See the [expanded timeouts guide](/tutorials/expanded-timeouts.md) for details on configuring all timeout values.

#### Configure TLS for the OpenFaaS dashboard

If you're using OpenFaaS Standard or OpenFaaS for Enterprises, you will want to expose the [OpenFaaS Dashboard](/openfaas-pro/dashboard.md) as well.

With Gateway API, both the Gateway and the HTTPRoute objects must include the desired hostname.

##### Add a listener to the Gateway

Add a second HTTPS listener for the dashboard domain to the existing Gateway:

```bash
export GW_DOMAIN="gw.example.com"
export DASHBOARD_DOMAIN="dashboard.example.com"

cat > gateway.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openfaas-gateway
  namespace: openfaas
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
    - name: gateway
      hostname: "$GW_DOMAIN"
      port: 443
      protocol: HTTPS
      allowedRoutes:
        namespaces:
          from: Same
      tls:
        mode: Terminate
        certificateRefs:
          - name: openfaas-gateway-cert
    - name: dashboard
      hostname: "$DASHBOARD_DOMAIN"
      port: 443
      protocol: HTTPS
      allowedRoutes:
        namespaces:
          from: Same
      tls:
        mode: Terminate
        certificateRefs:
          - name: openfaas-dashboard-cert
EOF
```

```bash
kubectl apply -f gateway.yaml
```

cert-manager will detect the new HTTPS listener and automatically create a second Certificate for the dashboard domain.

Create an A or CNAME record for `dashboard.example.com` pointing to the same external IP as the Gateway.

Verify both certificates are ready:

```sh
$ kubectl get certificate -n openfaas

NAME                      READY   SECRET                    AGE
openfaas-gateway-cert     True    openfaas-gateway-cert     10m
openfaas-dashboard-cert   True    openfaas-dashboard-cert   2m
```

##### Create the HTTPRoute for the dashboard

```bash
export DASHBOARD_DOMAIN="dashboard.example.com"

cat > httproute-dashboard.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openfaas-dashboard
  namespace: openfaas
spec:
  parentRefs:
    - name: openfaas-gateway
  hostnames:
    - "$DASHBOARD_DOMAIN"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      timeouts:
        request: 10m
      backendRefs:
        - name: dashboard
          port: 8080
EOF
```

```bash
kubectl apply -f httproute-dashboard.yaml
```

Once the Gateway, HTTPRoutes, and certificates are configured, proceed to [Verifying the installation](#verifying-the-installation).

### AWS EKS with Gateway API

This section explains how to expose the OpenFaaS gateway on AWS EKS using the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) with [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/).

There are two approaches:

* **[ALB (L7)](#alb-with-gateway-api)** - The AWS Load Balancer Controller provisions an Application Load Balancer directly. TLS is terminated at the ALB using certificates from AWS Certificate Manager (ACM). No additional proxy or cert-manager is needed.
* **[NLB (L4) with a Gateway API controller](#nlb-with-envoy-gateway)** - The AWS Load Balancer Controller provisions a Network Load Balancer (NLB) that handles L4 forwarding to a Gateway API conformant gateway controller, which then performs L7 routing and TLS termination using cert-manager and Let's Encrypt. In this guide we use [Envoy Gateway](https://gateway.envoyproxy.io/) as the gateway controller.

#### Pre-requisites

* AWS EKS cluster
* A domain name under your control, and access to create A or CNAME records
* [AWS Load Balancer Controller v3.0+](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) installed
    * For the [ALB approach](#alb-with-gateway-api): Gateway API feature gates must be enabled

#### Install the AWS Load Balancer Controller

Follow the [AWS documentation to install the AWS Load Balancer Controller using Helm](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html). The installation guide covers IAM configuration and the controller deployment.

By default, the AWS Load Balancer Controller will not listen to Gateway API CRDs. To enable support, specify the following feature gate(s) in the Helm install:

| Feature Gate | Purpose |
|---|---|
| `ALBGatewayAPI=true` | Enable L7 routing (HTTPRoute, GRPCRoute) via ALB |
| `NLBGatewayAPI=true` | Enable L4 routing (TCPRoute, UDPRoute, TLSRoute) via NLB |

For example, to enable both ALB and NLB Gateway API support:

```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "controllerConfig.featureGates.ALBGatewayAPI=true"
```

!!! note "Enable only what you need"
    If use the [ALB approach](#alb-with-gateway-api), enable `ALBGatewayAPI=true`. The [NLB with Envoy Gateway approach](#nlb-with-envoy-gateway) does not require the AWS Load Balancer Controller's Gateway API features and the `ALBGatewayAPI=true` can be omitted from the Helm install.

Once installed, verify the controller is running:

```sh
$ kubectl get deployment -n kube-system aws-load-balancer-controller

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           84s
```

#### ALB with Gateway API

In this setup, the AWS Load Balancer Controller provisions an Application Load Balancer (ALB) directly from Gateway API resources. The ALB handles both L7 routing and TLS termination using certificates from [AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html).

This is the simpler approach — no additional proxy, cert-manager, or Let's Encrypt configuration is needed.

##### Create an ACM certificate

The ALB uses AWS Certificate Manager (ACM) for TLS. Create a certificate in ACM that covers the domains you need, for example `gw.example.com` and `dashboard.example.com`. You can create ACM certificates via the [AWS Console](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html), the AWS CLI, Terraform, or CloudFormation.

The AWS Load Balancer Controller supports automatic certificate discovery. If you set the `hostname` on your Gateway listeners, the controller will find a matching ACM certificate (including wildcard certificates like `*.example.com`) without needing to provide the ARN. Alternatively, you can reference the certificate explicitly by ARN in a `LoadBalancerConfiguration` resource.

##### Create a GatewayClass

Create a `GatewayClass` that references the AWS Load Balancer Controller for ALB:

```bash
cat > gatewayclass.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: aws-alb
spec:
  controllerName: gateway.k8s.aws/alb
EOF
```

Apply it:

```bash
kubectl apply -f gatewayclass.yaml
```

Verify that the GatewayClass is accepted:

```sh
kubectl get gatewayclass

NAME      CONTROLLER              ACCEPTED
aws-alb   gateway.k8s.aws/alb     True
```

##### Create the Gateway

Create a `LoadBalancerConfiguration` and a Gateway with HTTPS listeners for each hostname. The controller will automatically discover the matching ACM certificate for each listener hostname.

The AWS Load Balancer Controller defaults to an `internal` scheme. Set `scheme: internet-facing` in the `LoadBalancerConfiguration` if the ALB should be publicly accessible.

```bash
cat > lb-config.yaml <<EOF
apiVersion: gateway.k8s.aws/v1beta1
kind: LoadBalancerConfiguration
metadata:
  name: openfaas-lb-config
  namespace: openfaas
spec:
  scheme: internet-facing
EOF
```

Reference the `LoadBalancerConfiguration` from the Gateway:

```bash
export GW_DOMAIN="gw.example.com"
export DASHBOARD_DOMAIN="dashboard.example.com"

cat > gateway.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openfaas-gateway
  namespace: openfaas
spec:
  gatewayClassName: aws-alb
  infrastructure:
    parametersRef:
      kind: LoadBalancerConfiguration
      name: openfaas-lb-config
      group: gateway.k8s.aws
  listeners:
    - name: https-gw
      protocol: HTTPS
      port: 443
      hostname: "$GW_DOMAIN"
      allowedRoutes:
        namespaces:
          from: Same
    - name: https-dashboard
      protocol: HTTPS
      port: 443
      hostname: "$DASHBOARD_DOMAIN"
      allowedRoutes:
        namespaces:
          from: Same
EOF
```

```bash
kubectl apply -f lb-config.yaml
kubectl apply -f gateway.yaml
```

##### Create TargetGroupConfigurations

The AWS Load Balancer Controller defaults to `targetType: instance`, which requires backend services to be of type `NodePort` or `LoadBalancer`. Since the OpenFaaS gateway and dashboard services use `ClusterIP`, you need to create `TargetGroupConfiguration` resources to set the target type to `ip` so the ALB can route directly to Pod IPs. See [AWS target group types](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-type) for more details on the differences between `instance` and `ip` target types.

Create a `TargetGroupConfiguration` for the gateway service:

```bash
cat > targetgroup-gateway.yaml <<EOF
apiVersion: gateway.k8s.aws/v1beta1
kind: TargetGroupConfiguration
metadata:
  name: gateway-tgconfig
  namespace: openfaas
spec:
  targetReference:
    kind: Service
    name: gateway
    group: ""
  defaultConfiguration:
    targetType: ip
EOF
```

```bash
kubectl apply -f targetgroup-gateway.yaml
```

If you are using the [OpenFaaS Dashboard](/openfaas-pro/dashboard.md), create a `TargetGroupConfiguration` for the dashboard service:

```bash
cat > targetgroup-dashboard.yaml <<EOF
apiVersion: gateway.k8s.aws/v1beta1
kind: TargetGroupConfiguration
metadata:
  name: dashboard-tgconfig
  namespace: openfaas
spec:
  targetReference:
    kind: Service
    name: dashboard
    group: ""
  defaultConfiguration:
    targetType: ip
EOF
```

```bash
kubectl apply -f targetgroup-dashboard.yaml
```

##### Create HTTPRoutes

Create an HTTPRoute for the OpenFaaS gateway:

```bash
export GW_DOMAIN="gw.example.com"

cat > httproute.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openfaas-gateway
  namespace: openfaas
spec:
  parentRefs:
    - name: openfaas-gateway
      sectionName: https-gw
  hostnames:
    - "$GW_DOMAIN"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      timeouts:
        request: 10s
      backendRefs:
        - name: gateway
          port: 8080
EOF
```

```bash
kubectl apply -f httproute.yaml
```

The `timeouts.request` field sets the maximum duration for the gateway to respond to an HTTP request. This value should be set to match the `gateway.writeTimeout` configured in the OpenFaaS Helm chart. See the [expanded timeouts guide](/tutorials/expanded-timeouts.md) for details on configuring all timeout values.

If you are using the [OpenFaaS Dashboard](/openfaas-pro/dashboard.md), create an additional HTTPRoute:

```bash
export DASHBOARD_DOMAIN="dashboard.example.com"

cat > httproute-dashboard.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openfaas-dashboard
  namespace: openfaas
spec:
  parentRefs:
    - name: openfaas-gateway
      sectionName: https-dashboard
  hostnames:
    - "$DASHBOARD_DOMAIN"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      timeouts:
        request: 10m
      backendRefs:
        - name: dashboard
          port: 8080
EOF
```

```bash
kubectl apply -f httproute-dashboard.yaml
```

##### Create DNS records

Check the Gateway status to find the ALB hostname:

```sh
kubectl get gateway -n openfaas openfaas-gateway

NAME                CLASS     ADDRESS                      PROGRAMMED    AGE
openfaas-gateway    aws-alb   xxx.elb.amazonaws.com        True          1m3s
```

Create **CNAME records** pointing your domains (`gw.example.com`, `dashboard.example.com`) to the ALB hostname.

#### NLB with Envoy Gateway

In this setup, the AWS Load Balancer Controller provisions a single Network Load Balancer (NLB) that handles L4 forwarding to the Envoy proxy, which then performs L7 routing and TLS termination using cert-manager and Let's Encrypt certificates.

Make sure the [AWS Load Balancer Controller](#install-the-aws-load-balancer-controller) is installed before proceeding.

[Install Envoy Gateway](#install-envoy-gateway) first so that the required CRDs are available, then create an `EnvoyProxy` resource with NLB annotations so the AWS Load Balancer Controller creates an internet-facing Network Load Balancer:

```bash
cat > envoyproxy-nlb-config.yaml <<EOF
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: nlb-proxy-config
  namespace: openfaas
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: LoadBalancer
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
          service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
EOF

kubectl apply -f envoyproxy-nlb-config.yaml
```

Then follow the [Gateway API with Envoy Gateway](#gateway-api-with-envoy-gateway) installation guide above. When creating the Gateway, add the `infrastructure.parametersRef` field to reference the NLB configuration:

```diff
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openfaas-gateway
  namespace: openfaas
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
spec:
  gatewayClassName: eg
+  infrastructure:
+    parametersRef:
+      group: gateway.envoyproxy.io
+      kind: EnvoyProxy
+      name: nlb-proxy-config
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
    - name: gateway
      hostname: "gw.example.com"
      port: 443
      protocol: HTTPS
      allowedRoutes:
        namespaces:
          from: Same
      tls:
        mode: Terminate
        certificateRefs:
          - name: openfaas-gateway-cert
```

This is the only difference from the standard Envoy Gateway setup. On AWS, you'll see an NLB hostname (ending in `.elb.amazonaws.com`) instead of an IP address when checking the Gateway status.

## Ingress

This section covers setting up TLS using a Kubernetes Ingress Controller. The general instructions use Traefik with cert-manager. If you are running on AWS EKS, see [Install Traefik with Network Load Balancer (NLB)](#aws-eks-install-traefik-with-nlb).

The Ingress Controller reads Ingress resources to configure routing rules and handle TLS termination:

```
    Internet (HTTPS)
           │
           ▼
     Public IP ─────> DNS: gw.example.com
           │
           ▼
  ┌─────────────────┐
  │    Ingress      │  IngressClass (e.g. traefik)
  │   Controller    │  :80, :443 (TLS terminates here)
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │    Ingress      │  Host: gw.example.com, Path: /
  │    Resource     │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ gateway Service │  :8080
  └────────┬────────┘
           │
           ▼
    OpenFaaS Gateway
```

### Setup an Ingress Controller

This guide uses Traefik as the primary example, however any Ingress Controller will work with OpenFaaS. If you use a different Ingress Controller, make sure to update the `ingressClassName` field in the Ingress resources shown in the following steps.

You can check which Ingress Classes are available in your cluster with:

```sh
kubectl get ingressclass
```

#### Install Traefik

Install Traefik with Helm:

```sh
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install --namespace=traefik traefik traefik/traefik \
  --create-namespace
```

See also: [Traefik installation](https://doc.traefik.io/traefik/getting-started/install-traefik/)

#### Install cert-manager

cert-manager is a Kubernetes operator maintained by the Cloud Native Computing Foundation (CNCF) which automates TLS certificate management.

To install cert-manager:

```sh
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

See also: [cert-manager installation](https://cert-manager.io/docs/installation/)

#### Configure cert-manager

You'll need to create an Issuer or ClusterIssuer for your cert-manager installation. This will tell cert-manager which domain it is operating on, and how to register an account for you.

The below will create an Issuer that only operates in the openfaas namespace, with a HTTP01 challenge. Note the ingress class specified in the HTTP01 challenge, this should match the class of your Ingress controller.

```bash
cat > issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: openfaas
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - selector: {}
      http01:
        ingress:
          class: traefik
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: openfaas
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - selector: {}
      http01:
        ingress:
          class: traefik
---

EOF
```

Apply the staging and production Issuers:

```bash
kubectl apply -f issuer.yaml
```

#### Create the required DNS records

You will need to create an A or CNAME record for your domain, pointing to the public IP address of your Ingress controller.

After installing Traefik, you'll see a new LoadBalancer service for traefik in the `traefik` namespace. You can find the public IP address with:

```sh
kubectl get svc/traefik -n traefik

NAME      TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                      AGE
traefik   LoadBalancer   10.43.87.4   18.136.136.18   80:31876/TCP,443:31706/TCP   28d
```

Take the IP address from the `EXTERNAL-IP` column and create an A record for your domain in your domain management software, or a CNAME record if you see a domain name in this field.

!!! note "AWS EKS"
    On EKS the `EXTERNAL-IP` field shows a hostname rather than an IP address. Create a **CNAME record** pointing your domain to the NLB hostname instead of an A record.

All users should create an entry for: `gateway.example.com` and then OpenFaaS dashboard users should create an additional record pointing at the same address for: `dashboard.example.com`.

#### Configure TLS for the OpenFaaS gateway

You can now configure the OpenFaaS gateway to use TLS by setting the following Helm values, you can save them in a file called `tls.yaml`:
 
```sh
export GW_DOMAIN="gw.example.com"

cat > tls.yaml <<EOF
ingress:
  enabled: true
  ingressClassName: traefik
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
  tls:
    - hosts:
        - $GW_DOMAIN
      secretName: openfaas-gateway-cert
  hosts:
  - host: $GW_DOMAIN
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

If you're using something other than Traefik, then change the `ingressClassName` field accordingly. Note that the `kubernetes.io/ingress.class` annotation is deprecated and should not be used.

The `cert-manager.io/issuer` annotation is used to pick between the staging and production Issuers for Let's Encrypt. If this is your first time working with cert-manager, you may want to use the staging issuer first to avoid running into rate limits if you have something misconfigured.

> Note: For extended timeouts beyond Traefik's defaults, see the [expanded timeouts guide](/tutorials/expanded-timeouts.md#load-balancers-ingress-and-service-meshes) for information on configuring Traefik's EntryPoint timeouts.

Now upgrade OpenFaaS via helm, use any custom values.yaml files that you have saved from a previous installation:

```sh
helm repo update && \
    helm upgrade --install openfaas openfaas/openfaas \
        --namespace openfaas \
        --values tls.yaml \
        --values values-custom.yaml
```

#### Configure TLS for the OpenFaaS dashboard

If you're using OpenFaaS Standard or OpenFaaS for Enterprises, you will probably want to create an additional Ingress record for the OpenFaaS dashboard.

Edit the previous example:

```sh
export GW_DOMAIN="gw.example.com"
export DASHBOARD_DOMAIN="dashboard.example.com"

cat > tls.yaml <<EOF
ingress:
  enabled: true
  ingressClassName: traefik
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
  tls:
    - hosts:
        - $GW_DOMAIN
      secretName: openfaas-gateway-cert
    - hosts:
        - $DASHBOARD_DOMAIN
      secretName: openfaas-dashboard-cert
  hosts:
  - host: $GW_DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway
            port:
              number: 8080
  - host: $DASHBOARD_DOMAIN
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

Once the Ingress and certificates are configured, proceed to [Verifying the installation](#verifying-the-installation).

### AWS EKS: Install Traefik with NLB

If you're running on AWS EKS, the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) can be used to provision a Network Load Balancer (NLB) for Traefik's LoadBalancer Service. Traefik still acts as the Ingress Controller and handles TLS termination with cert-manager, but the NLB provides the public endpoint.

Follow the general [Ingress with Traefik](#ingress-with-traefik) instructions above, but replace the [Install Traefik](#install-traefik) step with the instructions below. All other steps (cert-manager, DNS records, Helm values) remain the same.

#### Install the AWS Load Balancer Controller

Follow the [AWS documentation to install the AWS Load Balancer Controller using Helm](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html). The installation guide covers IAM configuration and the controller deployment.

See also: [AWS Load Balancer Controller documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)

Once installed, verify the controller is running:

```sh
$ kubectl get deployment -n kube-system aws-load-balancer-controller

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           84s
```

#### Install Traefik with NLB annotations

On EKS with the AWS Load Balancer Controller, Traefik's LoadBalancer Service needs the correct annotation so that the controller provisions an internet-facing NLB.

Install Traefik using Helm with the required annotation:

```sh
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install --namespace=traefik traefik traefik/traefik \
  --create-namespace \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-scheme=internet-facing"
```

The `service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing` annotation ensures the NLB is publicly accessible. Without it, the AWS Load Balancer Controller defaults to an `internal` scheme, which would prevent Let's Encrypt HTTP01 challenges from reaching your cluster.

Verify that Traefik is running and has an external address:

```sh
$ kubectl get svc -n traefik

NAME      TYPE           CLUSTER-IP      EXTERNAL-IP                  PORT(S)                      AGE
traefik   LoadBalancer   10.100.45.123   xxx.elb.amazonaws.com        80:31876/TCP,443:31706/TCP   60s
```

The `EXTERNAL-IP` field will show an NLB hostname (e.g. `xxx.elb.amazonaws.com`).

Now continue with the remaining steps of the ingress controller setup from [Install cert-manager](#install-cert-manager) onwards.

## Verifying the installation

First, check that the DNS records you created have taken effect. You can use `nslookup` or `dig` to check that the domain names resolve to the public address of your proxy or Gateway service.

```sh
nslookup gw.example.com
```

For Ingress, verify that the Ingress records have the desired domains in the "HOSTS" field:

```sh
kubectl get ingress -n openfaas
```

For Gateway API, verify that the Gateway is programmed and HTTPRoutes are accepted:

```sh
kubectl get gateway -n openfaas
kubectl get httproute -n openfaas
```

Next, check that the certificates have been issued and that they're ready:

```sh
kubectl get certificates -n openfaas -o wide
```

If you're still encountering issues, you can check the logs of the cert-manager controller:

```sh
kubectl logs -n cert-manager deploy/cert-manager
```

Log into the OpenFaaS gateway using its new URL:

```sh
export OPENFAAS_URL=https://gw.example.com

PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
echo -n $PASSWORD | faas-cli login --username admin --password-stdin

# List some functions:

faas-cli list
```

If you're using Identity and Access Management (IAM) for OpenFaaS, [see the SSO instructions instead](/openfaas-pro/sso/cli.md).

## Timeouts for synchronous invocations

Despite configuring OpenFaaS and your functions for [extended timeouts](/tutorials/expanded-timeouts.md), you may find that your Ingress Controller, Gateway proxy, or Cloud Load Balancer implements its own timeouts on connections. If you think you have everything configured correctly for OpenFaaS, but see a timeout at a very specific number such as 30s or 60s, then check the timeouts on your Ingress Controller, Gateway, or Load Balancer.

**Gateway API** - Timeouts can usually be configured directly on the HTTPRoute object using the `timeouts.request` field.

**Traefik Ingress** - Timeouts are typically configured at the EntryPoint level in the static configuration. See the [expanded timeouts guide](/tutorials/expanded-timeouts.md#load-balancers-ingress-and-service-meshes) for more details.

**Ingress Nginx (retired)** - This project is retired and should not be used for new installations. If you are still using Ingress Nginx, to extend a synchronous invocation beyond one minute, add the `nginx.ingress.kubernetes.io/proxy-read-timeout` annotation to your Ingress resource. This annotation is specified in seconds - for example, to extend the timeout to 30 minutes, use `nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"`.