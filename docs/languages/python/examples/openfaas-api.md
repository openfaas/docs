Use Python's `requests` library to interact with the [OpenFaaS REST API](/reference/rest-api/) and manage functions and namespaces programmatically — useful for CI/CD pipelines, functions that manage other functions, or building self-service platforms on top of OpenFaaS.

Use-cases:

* Custom deployment tooling and CI/CD automation
* Functions that manage other functions and namespaces programmatically
* Building self-service platforms on top of OpenFaaS

This example creates a namespace if it does not already exist, then deploys a function into it.

## Overview

handler.py:

```python
import os
import json
import requests

def handle(event, context):
    gateway = os.getenv("gateway_url", "http://gateway.openfaas:8080")
    password = read_secret("openfaas-password")
    auth = ("admin", password)

    body = json.loads(event.body)
    ns = body.get("namespace", "openfaas-fn")

    namespaces = requests.get(
        f"{gateway}/system/namespaces",
        auth=auth,
    ).json()

    if ns not in namespaces:
        r = requests.post(
            f"{gateway}/system/namespace/",
            json={"name": ns, "annotations": {"openfaas": "1"}},
            auth=auth,
        )
        if r.status_code not in (200, 201):
            return {
                "statusCode": r.status_code,
                "body": f"Failed to create namespace: {r.text}",
            }

    r = requests.put(
        f"{gateway}/system/functions",
        json={
            "service": body["name"],
            "image": body["image"],
            "namespace": ns,
        },
        auth=auth,
    )

    return {
        "statusCode": r.status_code,
        "body": r.text,
    }

def read_secret(name):
    with open("/var/openfaas/secrets/" + name, "r") as f:
        return f.read().strip()
```

requirements.txt:

```
requests
```

stack.yaml:

```yaml
functions:
  deploy-function:
    lang: python3-http
    handler: ./deploy-function
    image: ttl.sh/openfaas-examples/deploy-function:latest
    secrets:
    - openfaas-password
```

The `requests` package is pure Python, so the Alpine-based `python3-http` template works here.

- The gateway URL defaults to `http://gateway.openfaas:8080`, the in-cluster address when running on Kubernetes. Override it with the `gateway_url` environment variable if needed.
- The handler authenticates with HTTP Basic Auth using the `admin` username and the password read from the `openfaas-password` secret.
- Namespace creation is idempotent — the handler checks whether the namespace exists before attempting to create it.

Because this function can manage other functions and namespaces, its own endpoint should be protected. See [Add authentication](#add-authentication) for how to do this.

## Step-by-step walkthrough

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-http
faas-cli new --lang python3-http deploy-function \
  --prefix ttl.sh/openfaas-examples
```

The example uses the public [ttl.sh](https://ttl.sh) registry — replace the prefix with your own registry for production use.

Update `deploy-function/handler.py` and `deploy-function/requirements.txt` with the code from the overview above.

### Create the openfaas-password secret

The gateway admin password is stored in a Kubernetes secret called `basic-auth` during installation. Retrieve it and create an OpenFaaS function secret named `openfaas-password` so the function can access it at runtime:

```bash
PASSWORD=$(kubectl get secret -n openfaas basic-auth \
  -o jsonpath="{.data.basic-auth-password}" | base64 --decode)

faas-cli secret create openfaas-password --from-literal="$PASSWORD"
```

At runtime, the secret is mounted as a file under `/var/openfaas/secrets/` inside the function container.

### Deploy and invoke

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter deploy-function \
 --tag digest
```

Deploy the `env` function into a `staging` namespace. The namespace is created automatically if it does not exist:

```bash
curl -X POST http://127.0.0.1:8080/function/deploy-function \
  -H "Content-Type: application/json" \
  --data '{
    "name": "env",
    "image": "ghcr.io/openfaas/alpine:latest",
    "namespace": "staging"
  }'
```

### Add authentication

This function manages other functions and namespaces, so its endpoint should require authentication. You can implement authentication directly in the handler code, or use the [built-in function authentication provided by OpenFaaS IAM](/openfaas-pro/iam/function-authentication/).

#### Built-in function authentication with OpenFaaS IAM

OpenFaaS IAM provides built-in function authentication without any code changes. Set the `jwt_auth` environment variable on the function and configure a Role and Policy to control who can invoke it.

```yaml
functions:
  deploy-function:
    lang: python3-http
    handler: ./deploy-function
    image: ttl.sh/openfaas-examples/deploy-function:latest
    secrets:
    - openfaas-password
    environment:
      jwt_auth: "true"
```

The watchdog enforces authentication automatically — callers must present a valid function access token in the `Authorization: Bearer` header. The request will never reaches the function handler if the token is missing or invalid.

See [Function Authentication](/openfaas-pro/iam/function-authentication/) for how to configure Roles and Policies and obtain function access tokens.

#### Implement authentication in the handler

The handler can validate any credential that suits your use case — a pre-shared token, an API key, or any other scheme. The example below uses a pre-shared token stored as an OpenFaaS secret and compared against the `Authorization: Bearer` header — any request without a valid token is rejected before any API calls are made.

Add the `valid_bearer` helper and a token check at the top of the handler:

```diff
 import os
 import json
 import requests
 
 def handle(event, context):
+    token = read_secret("deploy-function-token")
+    if not valid_bearer(token, event.headers):
+        return {"statusCode": 401, "body": "Unauthorized"}
+
     gateway = os.getenv("gateway_url", "http://gateway.openfaas:8080")
     password = read_secret("openfaas-password")
     auth = ("admin", password)
@@ ...
 
+def valid_bearer(token, headers):
+    if "Authorization" not in headers:
+        return False
+    authz = headers["Authorization"]
+    if not authz.startswith("Bearer "):
+        return False
+    return authz.split(" ", 1)[1] == token
+
 def read_secret(name):
     with open("/var/openfaas/secrets/" + name, "r") as f:
         return f.read().strip()
```

Add the new secret to `stack.yaml`:

```diff
     secrets:
     - openfaas-password
+    - deploy-function-token
```

Generate and create the token secret:

```bash
faas-cli secret generate -o ./deploy-function-token.txt
faas-cli secret create deploy-function-token \
  --from-file=./deploy-function-token.txt
```

Redeploy the function and invoke it with the token:

```bash
faas-cli up \
 --filter deploy-function \
 --tag digest
```

```bash
curl -X POST http://127.0.0.1:8080/function/deploy-function \
  -H "Authorization: Bearer $(cat ./deploy-function-token.txt)" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "env",
    "image": "ghcr.io/openfaas/alpine:latest",
    "namespace": "staging"
  }'
```

A request without a valid token returns `401 Unauthorized`.
