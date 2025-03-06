# Using secrets

This page shows how to use secrets within your functions for API tokens, connection strings, passwords, and other confidential configuration.

There is a two-step process to using a secret. First you need to create the secret via API, then you need to bind that secret to a function and consume it within your code.

## Design

* Secrets can be created using the OpenFaaS REST API or [faas-cli](/cli/secrets/)
* Each function can consume zero to many secrets
* The same secret can be consumed by more than one function
* Secrets must exist in the cluster at deployment time

**Environment variables vs. files**

Secrets should never be set in environment variables, no matter how tempting it is. These are always visible within the OpenFaaS REST API and from within the Pod spec in Kubernetes. They are also often exposed via debug tools, logs, traces and may leak to unauthorized users.

Instead, all secrets are made available in the container file-system and should be read from the following location: `/var/openfaas/secrets/<secret-name>`. In the sample below we show how to create and consume a secret in a function.

> See also: [YAML reference: environmental variables](yaml.md).

**Kubernetes vs faasd**

For Kubernetes, secrets are stored [within the built-in secrets store](https://kubernetes.io/docs/concepts/configuration/secret/) within the cluster. Some managed Kubernetes services will also encrypt the data at rest, but you must check with your provider.

For faasd, secrets are created as plaintext files under `/var/lib/faasd-provider/secrets`. When you deploy a function, these secrets are bind-mounted into your container.

## Secrets with multiple keys or files

Let's explore an example where you have a function which needs to connect to two different databases. You will have two different connection strings, one for MongoDB and one for Postgresql as separate files under `/var/openfaas/secrets`:

* `/var/openfaas/secrets/mongo-connection.txt`
* `/var/openfaas/secrets/postgres-connection.txt`

When using `faas-cli` to create and manage secrets, you can only have one file or literal within each Kubernetes secret, so you'll create two secrets with different names:

```bash
faas-cli secret create mongo-connection \
  --from-file=mongo-connection.txt=./mongo-connection.txt

faas-cli secret create postgres-connection \
  --from-file=postgres-connection.txt=./postgres-connection.txt
```

> Note that `openfaas-fn` is a default value for the `--namespace` flag, you don't need to specify it with `faas-cli`.

Then in stack.yml, you'll need to add both `mongo-connection` and `postgres-connection` to the `secrets` section.

```yaml
functions:
  my-function:
    ...
    secrets:
      - mongo-connection
      - postgres-connection
```

With `kubectl`, you can have multiple files or literals within a single secret.

```bash
kubectl create secret generic -n openfaas-fn database-connections \
  --from-file=mongo-connection.txt=./mongo-connection.txt \
  --from-file=postgres-connection.txt=./postgres-connection.txt
```

You'll only need to add one secret to the `secrets` section in stack.yml.

```yaml
functions:
  my-function:
    ...
    secrets:
      - database-connections
```

## How to consume a secret

Create a new function with the `python3-http` template:

```bash
faas-cli template store pull python3-http

export OPENFAAS_PREFIX=ttl.sh/alexellis

faas-cli new --lang python3-http protected-api
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

## How to update a secret

If you need to update a secret for a function, you can use the `faas-cli secret update` command:

```bash
faas-cli secret update protected-api-token \
  --from-file=protected-api-token.txt
```

The kubelet component of Kubernetes monitors changes in secrets and updates the contents of the files mounted into any Function Pods. It can take anywhere between a few seconds and minutes for new secrets to be rolled out by the kubelet, there is no way to speed this up this process, other than restarting the Deployment or Pod for the Function, which is not recommended.

For faasd users, the file will be updated immediately.

In order to take advantage of update secrets, you should either:

* Read the secret from disk every time you require it
* Use an fsnotify library to watch for changes, and re-read the secret at that time.

## Automated secrets

There are various options for managing secrets in a more automated way, such as:

You can manage secrets through Git repositories using the [SealedSecrets project from Bitnami](https://github.com/bitnami-labs/sealed-secrets). This approach enables GitOps or Infrastructure as Code (IaaC) - a public key is used to encrypt your secret files and literal values, which is then decrypted by a controller in the cluster using a separate private key. The [SOPS project](https://github.com/getsops/sops) originally created by Mozilla provides an alterantive to SealedSecrets and can also be used to encrypt secrets for storage in Git.

Another popular option is to use AWS Secrets Manager, Hashicorp Vault directly, or a cloud-based keystore using the Open Source [External Secrets Operator](https://external-secrets.io/latest/). The External Secrets Operator will read secrets from cloud-based key stores and inject them into Kubernetes secrets, which can then be used as normal by functions.

