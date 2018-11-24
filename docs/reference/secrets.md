# Using secrets

This page shows how to use secrets within your functions for API tokens, passwords and similar.

Using secrets is a two step process. First we need to define the secret in your cluster and then you need to 'use' the secret to your function. You can find a simple example function [ApiKeyProtected in the OpenFaaS repo](https://github.com/openfaas/faas/tree/master/sample-functions/ApiKeyProtected-Secrets). When we deploy this function we provide a secret key that it uses to authenticate requests.

## Creating the secret

It is generally easiest to read your secret values from files. For our examples we have created a simple text file `~/secrets/secret_api_key.txt` that looks like

```txt
R^YqzKzSJw51K9zPpQ3R3N
```

Now we need to define the secret in the cluster.

### Define a secret in Kubernetes

In Kubernetes we can leverage the [secrets api](https://kubernetes.io/docs/concepts/configuration/secret/) to safely store our secret values

From the commandline use

```sh
kubectl create secret generic secret-api-key \
  --from-file=secret-api-key=~/secrets/secret_api_key.txt \
  --namespace openfaas-fn
```

Here we have explicitly named the key of the secret value so that when it is mounted into the function container, it will be named exactly `secret-api-key` instead of `secret_api_key.txt`.

### Define a secret in Docker Swarm

For sensitive value we can leverage the [Docker Swarm Secrets](https://docs.docker.com/engine/swarm/secrets/) feature to safely store our secret values.

From the command line use

```sh
docker secret create secret-api-key \
 ~/secrets/secret_api_key.txt
```

## Use the secret in your function

Secrets are mounted as files to `/var/openfaas/secrets` inside your function. Using secrets is as simple as adding code to read the value from `/var/openfaas/secrets/secret-api-key`.

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

## Deploy a function with secrets

Now, update your stack file to include the secret:

```yaml
  provider:
    name: faas
    gateway: http://localhost:8080

  functions:
    protectedapi:
      lang: Dockerfile
      skip_build: true
      image: functions/api-key-protected:latest
      secrets:
      - secret-api-key
```

and then deploy `faas-cli deploy -f ./stack.yaml`

Once the deploy is done you can test the function using the cli. The function is very simple, it reads the secret value that is mounted into the container for you and then returns a success or failure message based on if your header matches that secret value. For example,

```sh
faas-cli invoke protectedapi -H "X-Api-Key=R^YqzKzSJw51K9zPpQ3R3N"
```

Resulting in

```txt
Unlocked the function!
```

When you use the wrong api key,

```sh
faas-cli invoke protectedapi -H "X-Api-Key=thisiswrong"
```

You get

```txt
Access denied!
```
