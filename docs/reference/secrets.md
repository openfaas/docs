# Using secrets

This page shows how to use secrets within your functions for API tokens, connection strings, passwords, and other confidential configuration.

There is a two-step process to using a secret. First you need to create the secret via API, then you need to bind that secret to a function and consume it within your code.

## Design

* Secrets can be created using the OpenFaaS REST API or [faas-cli](/cli/secrets/)
* Each function can consume zero to many secrets
* The same secret can be consumed by more than one function
* Secrets must exist in the cluster at deployment time

**Environment variables vs. files**

Secrets should never be set in environment variables, no matter how tempting it is. These are always visible within the OpenFaaS REST API and from within the Pod spec in Kubernetes.

Instead, all secrets are made available in the container file-system and should be read from the following location: `/var/openfaas/secrets/<secret-name>`. In the sample below we show how to create and consume a secret in a function. 

The faas-cli can be used to manage secrets on Kubernetes and for faasd.

> See also: [YAML reference: environmental variables](yaml.md).

**Kubernetes vs faasd**

For Kubernetes, secrets are stored [within the built-in secrets store](https://kubernetes.io/docs/concepts/configuration/secret/) within the cluster. Some managed Kubernetes services will also encrypt the data at rest, but you must check with your provider.

For faasd, secrets are created as plaintext files under `/var/lib/faasd-provider/secrets`. When you deploy a function, these secrets are bind-mounted into your container.

## Example of using a secret

Create a new function with the `python3-http` template:

```bash
faas-cli template store pull python3-http

export OPENFAAS_PREFIX=ttl.sh/alexellis

faas-cli new --lang python3-http protected-api
mv protected-api.yml stack.yml
```

Create a secret called `protected-api-token`, it will be used to authenticate all requests made to the new `protected-api` function.

```bash
openssl rand -hex 16 > protected-api-token.txt
```

You can use `faas-cli` or `kubectl` to create your secrets.

```bash
faas-cli secret create protected-api-token \
  --from-file=protected-api-token.txt
```

Or use `kubectl`:

```bash
kubectl create secret generic protected-api-token \
  --from-file=protected-api-token.txt \
  --namespace openfaas-fn
```

> Note: secrets created with `faas-cli` can only contain one file, but secrets created with `kubectl` can contain multiple elements.

Next, edit the `stack.yml` file and add the following `secrets` section:

```yaml
functions:
  protected-api:
...
    secrets:
      - protected-api-token
```

Edit `protected-api/handler.py` and enter the following code:

```python
def get_secret(key):
    with open("/var/openfaas/secrets/{}".format(key)) as f:
        return f.read().strip()

def valid_bearer(token, headers):
    if not "Authorization" in headers:
        return False
    authz = headers["Authorization"]
    if not authz.startswith("Bearer "):
        return False

    bearer = authz.split(" ", 1)

    return bearer[1] == token

def handle(event, context):
    token = get_secret("protected-api-token")

    if not valid_bearer(token, event.headers):
        return {
           "statusCode": 401,
           "body": "Invalid authentication"
        }

    return {
        "statusCode": 200,
        "body": "Hello from OpenFaaS!"
    }
```

Test the function:

```bash
faas-cli up

curl -i http://127.0.0.1:8080/function/protected-api \
  -H "Authorization: Bearer invalid"

HTTP/2 401
Invalid authentication

curl -i http://127.0.0.1:8080/function/protected-api \
  -H "Authorization: Bearer $(cat ./protected-api-token.txt)"

HTTP/2 200
Hello from OpenFaaS
```

## Secrets and Infrastructure as Code (IaaC)

You can manage secrets through Git repositories using the [SealedSecrets project from Bitnami](https://github.com/bitnami-labs/sealed-secrets). This approach enables GitOps or Infrastructure as Code (IaaC) - a public key is used to encrypt your secret files and literal values, which is then decrypted by a controller in the cluster using a separate private key.

Another popular option is to use AWS Secrets Manager, without Git with the open source [External Secrets Operator](https://external-secrets.io/latest/).
