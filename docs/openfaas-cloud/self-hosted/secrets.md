## Secrets with OpenFaaS Cloud

There are two ways to bind to [OpenFaaS secrets](/reference/secrets) with OpenFaaS Cloud which apply when self-hosted.

### Create secrets manually

You can create secrets manually via `faas-cli secret create` or by using `kubectl`. These secrets will be available to users if the prefix of the secret matches the owner of the code being deployed, i.e.

If you are using an organization or repo named `myorg` and want a secret named `api-key` you could run:

```sh
$ faas-cli secret create myorg-api-key
```

In your `stack.yml` file in the `secrets` section, you could then reference the `api-key` secret.

This method relies on you having administrative access or making a request to your administrator.

### Use SealedSecrets in your repo

The preferred method available for Kubernetes uses SealedSecrets by Bitnami. We will use the `kubeseal` tool to encrypt our secrets using the public key of the cluster. These can then be placed in a file named `secrets.yaml` in your repo no matter whether it is public or private, your data will remain confidential.

#### Pre-reqs:

*  If you installed OpenFaaS Cloud using `ofc-bootstrap` then SealedSecrets will already be enabled, otherwise you can [follow the development guide](https://github.com/openfaas/openfaas-cloud/blob/master/docs/DEV.md#secrets).

* Follow [these instructions](https://github.com/openfaas/faas-cli#openfaas-cloud-extensions) to install `kubeseal` and to export the public key of your cluster.

> Note: If you are using the Community Cluster, then you can fetch the [public key here](https://github.com/openfaas/cloud-functions/blob/master/pub-cert.pem).

#### Seal a secret

See the [secrets](../secrets.md) documentation for information on how to use sealed secrets. 