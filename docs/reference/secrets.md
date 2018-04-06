# Using secrets

This page shows how to use secrets within your functions for API tokens, passwords and similar.

Using secrets is a two step process. First we need to define the secret in your cluster and then you need to 'use' the secret to your function. You can find a simple example function [ApiKeyProtected in the OpenFaaS repo](https://github.com/openfaas/faas/tree/master/sample-functions/ApiKeyProtected-Secrets). When we deploy this function we provide a secret key that it uses to authenticate requests.

_Note_: The examples in the following section require `faas-cli` version `>=0.5.1`

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
kubectl create secret generic secret_api_key --from-file=secret_api_key=~/secrets/secret_api_key.txt
```

Here we have explicitly named the key of the secret value so that when it is mounted into the function container, it will be named exactly `secret_api_key` instead of `secret_api_key.txt`.

### Define a secret in Docker Swarm

For sensitive value we can leverage the [Docker Swarm Secrets](https://docs.docker.com/engine/swarm/secrets/) feature to safely store our secret values.

From the command line use

```sh
docker secret create secret_api_key ~/secrets/secret_api_key.txt
```

## Use the secret in your function

This is the simplest part, update your stack file to include the secret:

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
          - secret_api_key
    ```

and then deploy `faas-cli deploy -f ./stack.yaml`

Once the deploy is done you can test the function using

```sh
faas-cli invoke protectedapi -H "X-Api-Key=R^YqzKzSJw51K9zPpQ3R3N"
```
