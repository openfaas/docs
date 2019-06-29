# Manage secrets

OpenFaaS Cloud sealed secrets lets you store and manage sensitive information securely

## Create

Secrets in OpenFaaS Cloud will be mounted as data volumes and made accessible to your function

To consume a **secret** in a function:

1.\ Create the sealed secret object `secrets.yml` using literals or from a file

`faas-cli cloud seal --cert pub.cert --name {username}-database-creds --literal username=admin --literal password=1234`

|                   |                         |
|-------------------|-------------------------|
| --name       | The name of the secret object prefixed with your GitHub username |
| --cert       | The OpenFaaS cloud [public key](https://github.com/openfaas/cloud-functions/blob/master/pub-cert.pem). Please download to a local file |
| --literal    | Secret key and value pair. You can specify this parameter more than once to add more secrets |
| --from-file  | Read secret from file. **Note** secret key name will be the filename. At the moment we do not support specifying the key name when creating a secret from a file. |


2.\ Update your function to include the secret in your `stack.yml`

```yaml
  my-function:   
    secrets:
      - database-creds
```

> **Note:** GitHub `{username}` prefix is removed in the `stack.yml`


3.\ Update your function code to read the secret from `/var/openfaas/secrets/{key}`

Where `{key}` is the name of the key pair. From our example in step 1 `username` or `password`:

- `/var/openfaas/secrets/username`
- `/var/openfaas/secrets/password` 

If you are writing your code in Go, the OpenFaaS Cloud SDK provides a [ReadSecret](https://github.com/openfaas/openfaas-cloud/blob/master/sdk/secrets.go) method.

4.\ Commit `secrets.yml` and other modified files. **Note** `secrets.yml` must be in the root of your GitHub repository

#### Troubleshooting

The steps above must be followed precisely and if you have mis-read any of the details this may result in the secret not being accessible.

Notes:

* When using `kubeseal` your secret name needs to be prefixed with your username i.e. `alexellis-my-secret`
* Each `--literal` must have no prefix, it's the exact name you will use in `stack.yml`
* In `stack.yml` your secrets should have no prefix, this is added later automatically
* You must commit `secrets.yaml` into your repo and do a `git push`

If in doubt check your results against [this reference repository](https://github.com/alexellis/my-fn).