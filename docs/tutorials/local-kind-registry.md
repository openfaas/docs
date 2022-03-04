# Use a local registry with KinD

A local registry can save on bandwidth costs and means your OpenFaaS functions don't leave your local computer when running `faas-cli up`

Not only is it much quicker, but it's also simple to configure if you're using KinD.

## Prerequisite:

You need to have **Docker** installed on your machine.

### Install arkade

We will use [arkade](https://github.com/alexellis/arkade) to install and deploy apps and services to Kubernetes.

```bash
# Download only, install yourself with sudo
$ curl -sLS https://get.arkade.dev | sh

# Download and install
$ curl -sLS https://get.arkade.dev | sudo sh
```

**arkade commands:**

* use `arkade get` to download CLI tools and applications.
* use `arkade install` to install applications using [helm charts](https://helm.sh/docs/topics/charts/) or vanilla YAML files.
* use `arkade info` to get back info about an app you've installed

### Install kubectl

[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) is a command line tool that talks to the Kubernetes API for performing actions on our cluster.

```bash
$ arkade get kubectl
```

## Create the KinD cluster with a local registry enabled

We will set up our local Kubernetes cluster using [**KinD**](https://github.com/kubernetes-sigs/kind) (Kubernetes in Docker). 

### Install KinD

These instructions are adapted from the KinD documentation. Our goal is to keep everything locally including a local Docker registry.

> The official KinD docs provides a [shell script](https://kind.sigs.k8s.io/docs/user/local-registry/) to create a Kubernetes cluster with local Docker registry enabled.

```bash
#!/bin/sh
set -o errexit

# create registry container unless it already exists
reg_name='kind-registry'
reg_port='5000'
running="$(docker inspect -f '{{.State.Running}}' "${reg_name}" 2>/dev/null || true)"
if [ "${running}" != 'true' ]; then
  docker run \
    -d --restart=always -p "${reg_port}:5000" --name "${reg_name}" \
    registry:2
fi

# create a cluster with the local registry enabled in containerd
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:${reg_port}"]
    endpoint = ["http://${reg_name}:${reg_port}"]
EOF

# connect the registry to the cluster network
docker network connect "kind" "${reg_name}"

# tell https://tilt.dev to use the registry
# https://docs.tilt.dev/choosing_clusters.html#discovering-the-registry
for node in $(kind get nodes); do
  kubectl annotate node "${node}" "kind.x-k8s.io/registry=localhost:${reg_port}";
done
```

> View the shell script in the [KinD docs](https://kind.sigs.k8s.io/examples/kind-with-registry.sh)

---

### Note:

You can find similar solutions for other local Kubernetes distributions:

* [k3d](https://k3d.io/usage/guides/registries/#using-a-local-registry) - local registries
* [minikube](https://minikube.sigs.k8s.io/docs/handbook/registry/) - registry add-on
* [microk8s](https://microk8s.io/docs/registry-built-in) - built-in registry
---

Make the script executable:

```bash
$ chmod +x kind-with-registry.sh
```

Run it to create your local cluster with registry:

```bash
$ ./kind-with-registry.sh
```

Make sure the `kubectl` context is set to the newly created cluster:

```bash
$ kubectl config current-context
```

If the result is not `kind-kind` then execute:

```bash
$ kubectl config use kind-kind
```

Make sure the cluster is running:

```bash
$ kubectl cluster-info
```

Make sure Docker registry is running.

```bash
$ docker logs -f kind-registry
```

### Deploy OpenFaaS

Deploy OpenFaaS and its CLI:

```bash
$ arkade install openfaas
$ arkade get faas-cli
```

Then log in and port-forward OpenFaaS using the instructions given, or run `arkade info openfaas` to get them a second time.

### Create a Function

We will take an example of a simple function; a dictionary that returns the meaning of word you query. We will be using the [PyDictionary](https://pypi.org/project/PyDictionary/) module for this setup.

Pull python language template from store:

```bash
$ faas-cli template store pull python3-flask
```

We will be using the [python3-flask-debian](https://github.com/openfaas-incubator/python-flask-template/tree/master/template/python3-flask-debian) template.

Setup your `OPENFAAS_PREFIX` variable to configure the address of your registry:

```bash
export OPENFAAS_PREFIX=localhost:5000
```

> Note: Docker for Mac users may need to change "localhost" to the IP address of their LAN or WiFi adapter as shown on `ifconfig` such as `192.168.0.14`

Create a new function using the template:

```bash
$ export FN=pydict
$ faas-cli new $FN --lang python3-flask-debian
```

This will create a directory for your function and a YAML config file with the function name you provided: 

* pydict/
* pydict.yml

Add dependency to the `pydict/requirements.txt` file: 

```txt
PyDictionary
```

Update `handler.py` with the following code.

```python
from PyDictionary import PyDictionary

dictionary = PyDictionary()

def handle(word):     
    return dictionary.meaning(word)
```

Our minimal function is complete. 

### Stack file

You will see that the OpenFaaS stack YAML file `pydict.yml` has `localhost:5000` in its image destination.

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  pydict:
    lang: python3-flask-debian
    handler: ./pydict
    image: localhost:5000/pydict:latest
```

## Build Push Deploy 

With our setup ready; we can now build our image, push it to the registry, and deploy it to Kubernetes. And using `faas-cli` it is possible with a single command! 

```bash
faas-cli up -f pydict.yml
```

## Test the function

We can invoke our function from CLI using `faas-cli` or `curl`.

```bash
$ echo "advocate" | faas-cli invoke pydict

{"Noun":["a person who pleads for a cause or propounds an idea","a lawyer who pleads cases in court"],"Verb":["push for something","speak, plead, or argue in favor of"]}
```

### Wrapping Up

Now that you have a local registry, you can speed up your local development of functions by keeping the container images within your local computer.

> This tutorial is based upon the KinD docs and [a post by Yankee Maharjan](https://dev.to/yankee/deploy-your-serverless-python-function-locally-with-openfaas-in-kubernetes-18jf).
