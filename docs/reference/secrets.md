# Using secrets

This page shows how to use secrets within your functions for API tokens, passwords and similar.

Using secrets is a two step process. First you need to define a new secret in your cluster and then you need to 'use' the secret to your function by adding it the deployment request or stack YAML file.

## Design

* Secrets can be specified via API, CLI or YAML file
* You can use one to many secrets in a function
* Secrets must exist in the cluster at deployment time
* You can create, list, delete and update secrets via the [faas-cli](/cli/secrets/).

### A note on environmental variables

All secrets are made available in the container file-system and should be read from the following location: `/var/openfaas/secrets/<secret-name>`. In the sample below we show how to create and consume a secret in a function. 

> Note: The OpenFaaS philosophy is that environment variables should be used for non-confidential configuration values only, and not used to inject secrets.

The faas-cli can be used to manage secrets on Kubernetes, faasd, and Swarm.

> See also: [YAML reference: environmental variables](yaml.md).

## Sample

We have built a sample function that can be deployed alongside a secret (an API key) to validate incoming requests. It is available in the [openfaas/faas](https://github.com/openfaas/faas/) repo: [ApiKeyProtected](https://github.com/openfaas/faas/tree/master/sample-functions/ApiKeyProtected-Secrets). Only requests presenting a valid API key value will be validated.

### Creating a file for the secret

Create a text file named `secret-api-key.txt` and add the following value:

```txt
R^YqzKzSJw51K9zPpQ3R3N
```

Now we can import the secret into the cluster.

#### Define the secret with `faas-cli`

```sh
faas-cli secret create secret-api-key \
  --from-file=secret-api-key.txt
```

> Note: only one key or file is supported when creating a Kubernetes secret with `faas-cli`, to use multiple keys or files in a single secret, see the next section.

You can create the secret with `faas-cli secret create`, or by using the Docker / Kubernetes CLI.

#### Define a secret in Kubernetes (advanced)

In Kubernetes we can leverage the [built-in secret store](https://kubernetes.io/docs/concepts/configuration/secret/) to securely store secrets for functions.

Type in:

```sh
kubectl create secret generic secret-api-key \
  --from-file=secret-api-key=secret-api-key.txt \
  --namespace openfaas-fn
```

Here we have explicitly named the key of the secret value so that when it is mounted into the function container, it will be named exactly `secret-api-key` instead of `secret_api_key.txt`.

You can skip creating a file and use input directly from the command-line like this:

```sh
kubectl create secret generic secret-api-key \
  --from-literal secret-api-key="R^YqzKzSJw51K9zPpQ3R3N" \
  --namespace openfaas-fn
```

#### Define a secret in Docker Swarm (advanced)

Docker has a built-in [secrets store](https://docs.docker.com/engine/swarm/secrets/) just like Kubernetes which can be used to securely store secrets for our functions.

Type in:

```sh
docker secret create secret-api-key \
 ~/secrets/secret_api_key.txt
```

or:

```sh
echo "R^YqzKzSJw51K9zPpQ3R3N" | docker secret create secret-api-key -
```

#### Define a secret in faasd (advanced)

For faasd, the secrets created for functions are held as files at `/var/lib/faasd-provider/secrets`. When you deploy a function, these secrets are bind-mounted into your container.

### Use the secret in your function

OpenFaaS secrets are mounted as files to `/var/openfaas/secrets` inside your function's filesystem. To use a secret, just read the file from the secrets location using the name of the secret for the filename such as: `/var/openfaas/secrets/secret-api-key`.

_Note_: prior to version `0.8.2` secrets were mounted to `/run/secrets`. The example functions demonstrate a smooth upgrade implementation.

A simple `go` implementation could look like this

```go
func getAPISecret(secretName string) (secretBytes []byte, err error) {
	// read from the openfaas secrets folder
	secretBytes, err = ioutil.ReadFile("/var/openfaas/secrets/" + secretName)
	if err != nil {
		// read from the original location for backwards compatibility with openfaas <= 0.8.2
		secretBytes, err = ioutil.ReadFile("/run/secrets/" + secretName)
	}

	return secretBytes, err
}
```

This example comes from the [`ApiKeyProtected`](https://github.com/openfaas/faas/tree/master/sample-functions/ApiKeyProtected-Secrets) sample function.

### Deploy a function with secrets

Create a `stack.yaml` file in the current directory:

```yaml
  provider:
    name: openfaas

  functions:
    protectedapi:
      lang: dockerfile
      skip_build: true
      image: functions/api-key-protected:latest
      secrets:
      - secret-api-key
```

Now deploy the function with: `faas-cli deploy`

Once the deploy is done you can test the function using the `faas-cli` or `curl`. The function reads the secret value that was mounted into the container by OpenFaaS and then returns a success or failure message based on if your header matches that secret value. The same code runs exactly the same without modifications on both Kubernetes and Docker Swarm.

Let's see how that works:

```sh
echo | faas-cli invoke protectedapi -H "X-Api-Key=R^YqzKzSJw51K9zPpQ3R3N"
Unlocked the function!
```

Now let's use an incorrect value for the api-key:

```sh
echo | faas-cli invoke protectedapi -H "X-Api-Key=thisiswrong"
Access denied!
```

You can also use multiple secrets for the same function or across multiple functions.

## Secrets in git

You can also manage secrets through Git repositories using the [SealedSecrets project from Bitnami](https://github.com/bitnami-labs/sealed-secrets).

This approach enables GitOps or Infrastructure as Code (IaaC) - a public key is used to encrypt your secret files and literal values, which is then decrypted by a controller in the cluster using a separate private key.

See also: [OpenFaaS Cloud](/openfaas-cloud/intro/).
