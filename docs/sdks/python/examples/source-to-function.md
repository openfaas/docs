Use the [OpenFaaS Python SDK](https://github.com/openfaas/python-sdk) and the [Function Builder API](/openfaas-pro/builder/) to go from source code to a running, authenticated function in a custom namespace, entirely from Python.

Use-cases:

* SaaS platforms where users supply their own code
* Multi-tenant environments where each tenant gets an isolated namespace
* CI/CD pipelines that go from source to a running function in a single script

The `greeter` function validates a Bearer token against a mounted secret. The orchestration script creates a namespace and secret, builds the function image from source, deploys it, and invokes the function once it is ready.

## Overview

The example consists of a `greeter` function and an orchestration script (`main.py`) that drives the full workflow.

greeter/handler.py:

```python
import json
import sys


def handle(event, context):
    if event.path == "/_/ready":
        return {"statusCode": 200, "body": "OK"}

    secret_path = "/var/openfaas/secrets/api-key"
    try:
        with open(secret_path) as f:
            api_key = f.read().strip()
    except OSError:
        print("Error: secret not available", flush=True, file=sys.stderr)
        return {
            "statusCode": 500,
            "body": json.dumps({"error": "Secret not available"}),
            "headers": {"Content-Type": "application/json"},
        }

    auth_header = event.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        print("Unauthorized: missing or malformed Authorization header", flush=True, file=sys.stderr)
        return {
            "statusCode": 401,
            "body": json.dumps({"error": "Unauthorized"}),
            "headers": {"Content-Type": "application/json"},
        }

    token = auth_header[len("Bearer "):]
    if token != api_key:
        print("Unauthorized: invalid token", flush=True, file=sys.stderr)
        return {
            "statusCode": 401,
            "body": json.dumps({"error": "Unauthorized"}),
            "headers": {"Content-Type": "application/json"},
        }

    print("Request authorized, returning greeting", flush=True)
    return {
        "statusCode": 200,
        "body": json.dumps({"message": "Hello from OpenFaaS!"}),
        "headers": {"Content-Type": "application/json"},
    }
```

main.py:

```python
import os
import sys
import time
import uuid

from openfaas_sdk import BasicAuth, Client
from openfaas_sdk.builder import BuildConfig, FunctionBuilder, create_build_context, make_tar
from openfaas_sdk.exceptions import APIConnectionError, ForbiddenError, NotFoundError, UnauthorizedError
from openfaas_sdk.models import FunctionDeployment, FunctionNamespace, Secret

GATEWAY_URL = os.environ.get("OPENFAAS_GATEWAY", "http://127.0.0.1:8080")
BUILDER_URL = os.environ.get("BUILDER_URL", "http://127.0.0.1:8081")
PAYLOAD_SECRET_PATH = os.environ.get("PAYLOAD_SECRET_PATH", "/var/secrets/payload-secret")

NAMESPACE = "tenant1"
FUNCTION_NAME = "greeter"
SECRET_NAME = "api-key"
IMAGE = "ttl.sh/greeter:1h"

def read_file(path):
    with open(path) as f:
        return f.read().strip()

def wait_for_ready(client, name, namespace, timeout=120):
    """Poll until at least one replica is available or the timeout is reached."""
    deadline = time.time() + timeout
    while time.time() < deadline:
        try:
            fn = client.get_function(name, namespace)
            if fn.available_replicas and fn.available_replicas >= 1:
                return
        except NotFoundError:
            pass
        time.sleep(3)
    print(f"Timed out waiting for {name!r} to become ready.", file=sys.stderr)
    sys.exit(1)

password = os.environ["OPENFAAS_PASSWORD"]
hmac_secret = read_file(PAYLOAD_SECRET_PATH)

try:
    with Client(gateway_url=GATEWAY_URL, auth=BasicAuth("admin", password)) as client:

        # Create an isolated namespace for the tenant.
        print(f"Creating namespace {NAMESPACE!r}")
        client.create_namespace(FunctionNamespace(name=NAMESPACE))

        # Generate a random API key and store it as an OpenFaaS secret.
        # The function reads this value at runtime from the mounted secret file.
        api_key = str(uuid.uuid4())
        print(f"Creating secret {SECRET_NAME!r}")
        client.create_secret(Secret(name=SECRET_NAME, namespace=NAMESPACE, value=api_key))

        # Assemble the Docker build context from the template and handler,
        # then pack it into a tar archive with the build configuration.
        print(f"Assembling build context for {FUNCTION_NAME!r}")
        context_path = create_build_context(
            function_name=FUNCTION_NAME,
            handler="./greeter",
            language="python3-http",
            template_dir="./template",
            build_dir="./build",
        )
        make_tar("/tmp/greeter.tar", context_path, BuildConfig(image=IMAGE, platforms=["linux/amd64"]))

        # Send the tar to the Function Builder and stream log lines as they arrive.
        print(f"Building {IMAGE!r}")
        builder = FunctionBuilder(BUILDER_URL, hmac_secret=hmac_secret)
        for result in builder.build_stream("/tmp/greeter.tar"):
            for line in result.log:
                print(line)
            if result.status == "failed":
                print("Build failed.", file=sys.stderr)
                sys.exit(1)

        # Deploy the function into the tenant namespace with the secret mounted.
        # The readiness annotation tells Kubernetes to use the function's /_/ready
        # endpoint so the pod is only marked ready once the handler is fully started.
        print(f"Deploying {FUNCTION_NAME!r} into {NAMESPACE!r}")
        client.deploy(FunctionDeployment(
            service=FUNCTION_NAME,
            image=IMAGE,
            namespace=NAMESPACE,
            secrets=[SECRET_NAME],
            annotations={
                "com.openfaas.ready.http.path": "/_/ready",
            },
        ))

        print("Waiting for function to become ready")
        wait_for_ready(client, FUNCTION_NAME, NAMESPACE)

        # Invoke the function with the generated API key as a Bearer token.
        print("Invoking function")
        resp = client.invoke_function(
            FUNCTION_NAME,
            namespace=NAMESPACE,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        print(resp.status_code, resp.text)

        # Get the last 20 log lines from the function.
        print("Fetching logs")
        for msg in client.get_logs(FUNCTION_NAME, NAMESPACE, tail=20):
            print(f"[{msg.timestamp}] {msg.instance}: {msg.text}")

except UnauthorizedError:
    print("Unauthorized. Check your OPENFAAS_PASSWORD.", file=sys.stderr)
    sys.exit(1)
except (ForbiddenError, APIConnectionError) as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

requirements.txt:

```
git+https://github.com/openfaas/python-sdk.git
```

- A random UUID is generated as the API key at runtime and stored as an OpenFaaS secret.
- The handler responds to `/_/ready` with a 200 so Kubernetes only marks the pod ready once the Flask server is fully listening. The `com.openfaas.ready.http.path` annotation configures OpenFaaS to use this endpoint for the readiness probe.
- `create_build_context` assembles a Docker build context from the OpenFaaS template and handler directory. `make_tar` packs it with the build configuration into a tar archive ready for the Function Builder. `build_stream` sends the tar and yields log lines as they arrive.

## Step-by-step walkthrough

### Prerequisites

- Python 3.10+
- [`faas-cli`](https://github.com/openfaas/faas-cli) installed
- OpenFaaS with the [Function Builder](/openfaas-pro/builder/) enabled
- A container registry the builder can push to. This example uses [ttl.sh](https://ttl.sh) (no credentials required)

### Set up the project

Create a directory for the example and add the files from the overview above:

```bash
mkdir source-to-function && cd source-to-function
```

Pull the `python3-http` template:

```bash
faas-cli template store pull python3-http
```

Scaffold the `greeter` function handler:

```bash
faas-cli new --lang python3-http greeter
```

Replace `greeter/handler.py` with the handler from the overview. The generated `requirements.txt` in the `greeter` directory can be left empty.

Create `main.py` in the project root with the script from the overview.

Make sure the OpenFaaS SDK is installed:

```bash
pip install git+https://github.com/openfaas/python-sdk.git
```

### Configure environment variables

| Variable | Default | Description |
|---|---|---|
| `OPENFAAS_PASSWORD` | — | **Required.** OpenFaaS gateway admin password |
| `OPENFAAS_GATEWAY` | `http://127.0.0.1:8080` | Gateway URL |
| `BUILDER_URL` | `http://127.0.0.1:8081` | Function Builder URL |
| `PAYLOAD_SECRET_PATH` | `/var/secrets/payload-secret` | Path to the builder HMAC payload secret |

Retrieve the gateway password and builder payload secret from the cluster:

```bash
export OPENFAAS_PASSWORD=$(kubectl get secret -n openfaas basic-auth \
  -o jsonpath="{.data.basic-auth-password}" | base64 --decode)

kubectl get secret -n openfaas payload-secret \
  -o jsonpath='{.data.payload-secret}' | base64 --decode \
  | sudo tee /var/secrets/payload-secret
```

### Run the script

Port-forward the gateway and builder, then run `main.py`:

```bash
kubectl port-forward -n openfaas svc/gateway 8080:8080 &
kubectl port-forward -n openfaas deploy/pro-builder 8081:8080 &

python main.py
```
