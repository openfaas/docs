# Manage secrets

The OpenFaaS CLI allows you to create, update, list and delete secrets using `faas-cli` instead of Docker or Kubernetes command line tools.
The reason behind this is to give you simplicity when you need to use secrets for your functions as well as to provide a layer of abstraction, as it will work for both Kubernetes and Docker Swarm.

It's available in those versions or later:
* faas-cli 0.8.3
* faas-swarm 0.6.1
* faas-netes 0.7.0
* openfaas-operator 0.9.1


## Create

To create a secret from stdin, you can run:

```
faas-cli secret create secret-name
```
or use pipe instead:
```
cat 04385e5c413c10ed68afb010ebe8c5dd706aa20a | faas-cli secret create secret-name
cat ~/Downloads/derek.pem | faas-cli secret create secret-name
```

If you want to pass a value then do:
```
faas-cli secret create secret-name --from-literal="04385e5c413c10ed68afb010ebe8c5dd706aa20a"
```

To create it from file use:
```
faas-cli secret create secret-name --from-file=~/Downloads/derek.pem
```

You can pass `--gateway` flag if you'd like to create the secret for a specific OpenFaaS instance.

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

## List

To list secrets for an OpenFaaS instance use:
```
faas-cli secret list --gateway http://127.0.0.1:8080
```
or
```
faas-cli secret ls --gateway http://127.0.0.1:8080
```
If you have set $OPENFAAS_URL you can use only 
```
faas-cli secret ls
```

This will output:
```
NAME
secret-name1
secret-name2
...
```

## Delete

You can delete secrets with:
```
faas-cli secret remove secret-name
```
or
```
faas-cli secret remove secret-name --gateway=http://127.0.0.1:8080
```
