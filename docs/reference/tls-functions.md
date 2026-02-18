## TLS for OpenFaaS Functions

When you expose the [OpenFaaS Gateway via TLS](/reference/tls-openfaas), each function is already accessible over TLS by adding `/function/NAME` to the URL.

This page is for users who want to create custom domains for individual functions, for example to access a function called `env` via `https://env.fn.example.com` instead of `https://gw.example.com/function/env`.

It's important that you never expose a function's Pod or Deployment directly, but only via the OpenFaaS gateway. The gateway is required in the invocation path to provide metrics, auto-scaling and asynchronous invocations. To expose a function on its own domain, you create an HTTPRoute that rewrites the path and forwards traffic to the OpenFaaS gateway service.

### Pre-requisites

* The [Kubernetes Gateway API](/reference/tls-openfaas#gateway-api) must already be set up for the OpenFaaS gateway, with a working Gateway, cert-manager Issuer. Follow the [TLS for OpenFaaS](/reference/tls-openfaas#gateway-api) guide first.
* A DNS record for the function's hostname (e.g. `env.fn.example.com`) pointing to the same external IP or hostname as the Gateway.

### How it works

Each function exposed on a custom domain needs two things:

1. **A listener on the Gateway** — for the function's hostname, so TLS can be terminated and a certificate provisioned by cert-manager.
2. **An HTTPRoute** — to match requests for the hostname and rewrite the path to `/function/NAME/` before forwarding to the OpenFaaS gateway service.

```
    Internet (HTTPS)
           │
           ▼
     Public IP ─────> DNS: env.fn.example.com
           │
           ▼
  ┌─────────────────┐
  │    Gateway      │  Listener: env.fn.example.com :443
  │                 │  TLS terminates here
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │   HTTPRoute     │  Host: env.fn.example.com
  │                 │  Rewrite: / → /function/env/
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

### Add a listener to the Gateway

Each function domain requires its own HTTPS listener on the Gateway. This is how cert-manager knows to provision a TLS certificate for the hostname.

Add a new listener to your existing Gateway. For example, to expose a function called `env` on `env.fn.example.com`:

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
+    - name: env
+      hostname: "env.fn.example.com"
+      port: 443
+      protocol: HTTPS
+      allowedRoutes:
+        namespaces:
+          from: Same
+      tls:
+        mode: Terminate
+        certificateRefs:
+          - name: env-openfaas-fn-cert
```

Apply the updated Gateway:

```bash
kubectl apply -f gateway.yaml
```

cert-manager will detect the new HTTPS listener and automatically create a Certificate for `env.fn.example.com`.

Create an A or CNAME record for `env.fn.example.com` pointing to the same external IP as the Gateway.

!!! note
    If you are exposing many functions, consider using a wildcard certificate (e.g. `*.fn.example.com`) and a wildcard DNS record to avoid creating individual records for each function.

### Create the HTTPRoute

The HTTPRoute rewrites incoming requests from `/` to `/function/NAME/` before forwarding them to the OpenFaaS gateway service. This ensures the request path matches what the OpenFaaS gateway expects.

!!! warning "Envoy Gateway and Istio require a workaround"
    The standard Gateway API `URLRewrite` filter with `ReplacePrefixMatch` does not behave consistently across all implementations. Envoy Gateway and Istio produce incorrect rewrites when the matched prefix is `/`. If you are using a different implementation such as Traefik use the **Standard Gateway API** tab. If you are using Envoy Gateway, use the **Envoy Gateway** tab which uses a regex-based rewrite to work around the issue.

=== "Envoy Gateway"

    Envoy Gateway (and Istio) have inconsistent behaviour with the standard `URLRewrite` `ReplacePrefixMatch` filter — when the matched prefix is `/` and the request path is also `/foo`, the rewrite produces an invalid function url (e.g. `/function/envfoo` instead of `/function/env/foo`). 

    To work around this, use an Envoy Gateway `HTTPRouteFilter` with a regex-based path rewrite instead:

    ```bash
    export FN_NAME="env"
    export FN_DOMAIN="env.fn.example.com"

    cat > httproute-fn-${FN_NAME}.yaml <<EOF
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: ${FN_NAME}
      namespace: openfaas
    spec:
      parentRefs:
        - name: openfaas-gateway
          sectionName: ${FN_NAME}
      hostnames:
        - "$FN_DOMAIN"
      rules:
        - matches:
            - path:
                type: PathPrefix
                value: /
          filters:
            - type: ExtensionRef
              extensionRef:
                group: gateway.envoyproxy.io
                kind: HTTPRouteFilter
                name: openfaas-${FN_NAME}-prefix
          timeouts:
            request: "21m"
          backendRefs:
            - name: gateway
              port: 8080
    ---
    apiVersion: gateway.envoyproxy.io/v1alpha1
    kind: HTTPRouteFilter
    metadata:
      name: openfaas-${FN_NAME}-prefix
      namespace: openfaas
    spec:
      urlRewrite:
        path:
          type: ReplaceRegexMatch
          replaceRegexMatch:
            pattern: '^/(.*)$'
            substitution: '/function/${FN_NAME}/\1'
    EOF
    ```

    ```bash
    kubectl apply -f httproute-fn-${FN_NAME}.yaml
    ```

    The `HTTPRouteFilter` is an Envoy Gateway extension CRD that supports regex-based path rewrites. The regex `^/(.*)$` captures the entire request path, and the substitution prepends `/function/env/` while preserving any sub-path.

=== "Standard Gateway API"

    The standard Gateway API `URLRewrite` filter with `ReplacePrefixMatch` works correctly with most implementations like Traefik:

    ```bash
    export FN_NAME="env"
    export FN_DOMAIN="env.fn.example.com"

    cat > httproute-fn-${FN_NAME}.yaml <<EOF
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: ${FN_NAME}
      namespace: openfaas
    spec:
      parentRefs:
        - name: openfaas-gateway
          sectionName: ${FN_NAME}
      hostnames:
        - "$FN_DOMAIN"
      rules:
        - matches:
            - path:
                type: PathPrefix
                value: /
          filters:
            - type: URLRewrite
              urlRewrite:
                path:
                  type: ReplacePrefixMatch
                  replacePrefixMatch: /function/${FN_NAME}/
          timeouts:
            request: "21m"
          backendRefs:
            - name: gateway
              port: 8080
    EOF
    ```

    ```bash
    kubectl apply -f httproute-fn-${FN_NAME}.yaml
    ```

The `parentRefs.sectionName` must match the name of the listener you added to the Gateway for this function. The `timeouts.request` field sets the maximum duration for a synchronous function invocation — adjust it to match your function's expected execution time.

### Verify the setup

Check that the certificate for the function domain has been issued:

```sh
kubectl get certificate -n openfaas

NAME                      READY   SECRET                    AGE
openfaas-gateway-cert     True    openfaas-gateway-cert     30m
env-openfaas-fn-cert      True    env-openfaas-fn-cert      2m
```

Verify the HTTPRoute is accepted:

```sh
kubectl get httproute -n openfaas

NAME                 HOSTNAMES                  AGE
openfaas-gateway     ["gw.example.com"]         30m
env-openfaas-fn      ["env.fn.example.com"]     2m
```

Invoke the function using its custom domain:

```sh
curl -s https://env.fn.example.com
```

The function is still accessible at its original path as well:

```sh
curl -s https://gw.example.com/function/env
```

### Exposing multiple functions

Repeat the steps above for each function you want to expose. For each function:

1. Add a new listener to the Gateway with the function's hostname
2. Create a DNS record for the hostname
3. Create an HTTPRoute (with the appropriate URL rewrite for your Gateway API implementation)

For example, to expose a second function called `sleep` on `sleep.fn.example.com`, add another listener to the Gateway:

```diff
   listeners:
     # ... existing listeners ...
+    - name: sleep
+      hostname: "sleep.fn.example.com"
+      port: 443
+      protocol: HTTPS
+      allowedRoutes:
+        namespaces:
+          from: Same
+      tls:
+        mode: Terminate
+        certificateRefs:
+          - name: sleep-openfaas-fn-cert
```

Then create an HTTPRoute following the same pattern as above, replacing the function name and domain.

