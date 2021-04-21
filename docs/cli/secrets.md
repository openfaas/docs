# Manage secrets

The OpenFaaS CLI allows you to create, update, list and delete secrets using `faas-cli` instead of Docker or Kubernetes command line tools.

The reason behind this is to give you simplicity when you need to use secrets for your functions as well as to provide a layer of abstraction, as it will work for both Kubernetes and [faasd](/deployment/faasd/).

See also: `faas-cli secret --help`

## Create

To create a secret from stdin, you can run:

```bash
faas-cli secret create secret-name
```

Or pipe the value from a file instead:

```
cat 04385e5c413c10ed68afb010ebe8c5dd706aa20a | faas-cli secret create secret-name
cat ~/Downloads/derek.pem | faas-cli secret create secret-name
```

If you want to pass a value then do:

```bash
faas-cli secret create secret-name \
  --from-literal="04385e5c413c10ed68afb010ebe8c5dd706aa20a"
```

To create it from file use:

```bash
faas-cli secret create secret-name \
  --from-file=~/Downloads/derek.pem
```

Target a specific namespace:

```bash
faas-cli secret create \ 
  --namespace staging-fn \
  secret-name \
  --from-literal="04385e5c413c10ed68afb010ebe8c5dd706aa20a"
```

Note: you will need to follow the instructions for [multiple namespaces](/reference/namespaces) for the above command.

You can pass `--gateway` flag if you'd like to create the secret for a specific OpenFaaS instance.

See also: `faas-cli secret create --help`

## Update

From stdin:
```
faas-cli secret update secret-name
```
or
```
cat 04385e5c413c10ed68afb010ebe8c5dd706aa20a | faas-cli secret update secret-name
cat ~/Downloads/derek.pem | faas-cli secret update secret-name
```

From literal:
```
faas-cli secret update secret-name --from-literal="04385e5c413c10ed68afb010ebe8c5dd706aa20a"
```

From file:
```
faas-cli secret update secret-name --from-file=~/Downloads/derek.pem
```

See also: `faas-cli secret update --help`

## List

To list secrets in the default namespace:

```bash
faas-cli secret list
faas-cli secret ls
```

To list secrets in an alternative namespace:

```bash
faas-cli secret ls --namespace staging
```

To list secrets for an OpenFaaS instance use:

```bash
faas-cli secret ls --gateway http://127.0.0.1:8080
```

If you have set `$OPENFAAS_URL` you can use only

```bash
faas-cli secret ls
```

This will output:

```bash
NAME
secret-name1
secret-name2
...
```

See also: `faas-cli secret list --help`

## Delete

You can delete secrets with:
```bash
faas-cli secret remove secret-name
```

or

```bash
faas-cli secret remove secret-name \
  --gateway=http://127.0.0.1:8080
```

See also: `faas-cli secret delete --help`
