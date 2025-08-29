# The Function Store

The Function Store makes it easy to share pre-built functions with other users.

The project maintains a number of official functions for testing and exploration, plus some contributions from the community.

## Find and explore functions in the store

To list available functions run:

```bash
$ faas-cli store list

FUNCTION          AUTHOR       DESCRIPTION
nodeinfo          openfaas     NodeInfo
env               openfaas     env
sleep             openfaas     sleep
shasum            openfaas     shasum
figlet            openfaas     figlet
printer           openfaas     printer
curl              openfaas     curl
external-ip       openfaas     external-ip
markdown          openfaas     markdown
youtube-dl        openfaas     youtube-dl
sentimentanalysis openfaas     SentimentAnalysis
hey               openfaas     hey
nslookup          openfaas     nslookup
certinfo          stefanprodan SSL/TLS cert info
nvidia-smi        openfaas     nvidia-smi
alpine            openfaas     alpine
cows              openfaas     ASCII Cows
```

For a given function say `env`, you can find more information by describing it:

```bash
$ faas-cli store describe env

Title:       env
Author:      openfaas
Description: 
Print the environment variables present in the function and HTTP request

Image:    ghcr.io/openfaas/alpine:latest
Process:  env
Repo URL: https://github.com/openfaas/store-functions
```

Certain functions have a minimum memory requirement, or set additional timeout values.

Try to describe something like the `inception` model:

```bash
faas-cli store describe inception --platform x86_64
Title:       Inception
Author:      alexellis
Description: 
This is a forked version of the work by Magnus Erik Hvass Pedersen - it has been
re-packaged as an OpenFaaS serverless function.

Image:    alexellis/inception:2019-02-17
Process:  python3 index.py
Repo URL: https://github.com/faas-and-furious/inception-function
Environment:
-  read_timeout:  60s
-  write_timeout: 60s
```

You'll notice it's not available for Arm or Apple Silicon, so I specified a `--platform` directly.

In this case a minimum timeout is set of 60s.

## Deploy a function from the store

To deploy a function as it is, simply run:

```bash
faas-cli store deploy nodeinfo
```

Then invoke it via `faas-cli invoke`, or obtain its URL on the gateway via `faas-cli describe nodeinfo`.

You can set additional labels, annotations, environment variables, and memory/CPU constraints by passing additional flags to the `store deploy command`.

For instance, to enable scale to zero:

```bash
faas-cli store deploy nodeinfo \
    --label com.openfaas.scale.zero=true \
    --label com.openfaas.scale.zero-duration=5m
```

## Create your own custom function store

You can create your own custom function store by creating a new Git repository, and placing a functions.json file in the root.

Then pass the `--url` flag to instruction the CLI to use your store instead of the official one.

```bash
faas-cli store list \
  --url https://raw.githubusercontent.com/openfaas/store/refs/heads/master/functions.json
```

When using GitHub to store your functions.json file make sure you specify the *raw* URL.

From `faas-cli` version 0.17.8 and higher, to avoid repeating the `--url` flag on each command, you can also set the `OPENFAAS_STORE` environment variable.

For convenience, if you want to persist the change, introduce the following in your .zshrc or .bashrc file:

```bash
export OPENFAAS_STORE=https://raw.githubusercontent.com/openfaas/store/refs/heads/master/functions.json
```
