# Take your Deployment Local

In this tutorial, you will learn to setup a local deployment environment for your functions from scratch. We will setup: 

1. Kubernetes 
2. Docker Registry 
3. OpenFaas

### Prerequisite:

You need to have **Docker** installed on your machine. The rest we will setup as we go along.

### Install arkade

We will use [arkade](https://github.com/alexellis/arkade) to install and deploy apps and services to Kubernetes.

```bash
$ curl -sLS [https://dl.get-arkade.dev](https://dl.get-arkade.dev) | sudo sh
```

**arkade commands:**

* use `arkade get` to download CLI tools and applications.

* use `arkade install` to install applications using [helm charts](https://helm.sh/docs/topics/charts/) or vanilla YAML files.

### Install kubectl

[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) is a command line tool that talks to the Kubernetes API for performing actions on our cluster.

```bash
$ arkade get kubectl

# follow the instructions on the screen.
```

## Creating a local Kubernetes cluster

We will set up our local Kubernetes cluster using [**KinD**](https://github.com/kubernetes-sigs/kind)(Kubernetes in Docker).

### Install KinD

Our goal is to keep everything local; this includes a local Docker registry ([Dockerhub](https://hub.docker.com/) for your local machine). KinD provides a [shell script](https://kind.sigs.k8s.io/docs/user/local-registry/) to create a Kubernetes cluster with local Docker registry enabled.

>  Download the shell script [here](https://kind.sigs.k8s.io/examples/kind-with-registry.sh).

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
OR 
$ docker ps -l
```

### Deploying OpenFaas to KinD cluster

We have our local Kubernetes cluster up and running. Next we deploy OpenFaas to support our functions.

```bash
$ arkade install openfaas
```

Make sure OpenFaas is deployed. It may take a minute or two.

```bash
$ kubectl get pods -n openfaas
```

Install `faas-cli`: 

```bash
$ arkade get faas-cli
```

### Create a Function

We will take an example of a simple function; a dictionary that returns the meaning of word you query. We will be using the [PyDictionary](https://pypi.org/project/PyDictionary/) module for this setup.

Pull python language template from store:

```bash
$ faas-cli template store pull python3-flask
```

This will pull all the templates for python from OpenFaas store to your project directory.

* project-directory/template

 We will be using the [python3-flask-debian](https://github.com/openfaas-incubator/python-flask-template/tree/master/template/python3-flask-debian) template. You can ignore the rest or delete them.

Create a new function using the template.

```bash
$ faas-cli new <function-name> --lang <language-template-name>

$ faas-cli new pydict --lang python3-flask-debian
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

### YAML configuration

The YAML config file tells the `faas-cli` about the functions and images we want to deploy, language templates we want to use, scaling factor, and other configurations for Kubernetes.

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  pydict:
    lang: python3-flask-debian
    handler: ./pydict
    image: pydict:latest

```

To push image to local registry, we have to update the `image` node to specify the registry IP address. Update value of `image` node as follows:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  pydict:
    lang: python3-flask-debian
    handler: ./pydict
    image: localhost:5000/pydict:latest # updated

```

### Port-forward to Localhost

We need to port-forward the OpenFaas gateway service inside the cluster to our local machine. This makes deployment and access to our functions possible.

Check for OpenFaas `gateway` service:

```bash
$ kubectl get service -n openfaas
```

Port-forward gateway service to your local machine.
```bash
$ kubectl port-forward -n openfaas svc/gateway 8080:8080
```

Now open a new terminal window, and prepare for take off!

## Build Push Deploy 

With our setup ready; we can now build our image, push it to the registry, and deploy it to Kubernetes.

**Build the image**:

```bash
$ faas-cli build -f pydict.yml
```

**Push the image to local registry**:

```bash
$ faas-cli push -f pydict.yml
```

Before we deploy the function, `faas-cli` requires authentication. 

**Generate password**:

```bash
$ PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
```

**Check for password:**

```bash
$ echo $PASSWORD
```

**Login to** `faas-cli`:

```bash
$ echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```

**Deploy the function**:

```bash
$ faas-cli deploy -f pydict.yml --gateway http://127.0.0.1:8080
```

**Make sure the function is running:**

```bash
$ kubectl get pods -n openfaas-fn
```

## Testing the function

### Test using CLI

We can invoke our function from CLI using `faas-cli` or `curl`.

```bash
$ echo "brevity" | faas-cli invoke pydict
```

### Test using OpenFaas Portal

Navigate to `localhost:8080` on your browser. A prompt will ask you for username and password. Use **admin** for *username* and for *password*, use the password generated earlier.

![OpenFaas UI](https://i.imgur.com/FjbjtuT.png)

### Wrapping Up

With this setup you can build and deploy your functions locally. If you want to create more functions:

1. scaffold a new function using `faas-cli new`

1. write your function

1. deploy it! - `faas-cli up -f <config-yaml-file>`

`faas-cli up` is a shortcut for `build, push and deploy` command.

> Please show support and **Star** the project on [Github](https://github.com/openfaas/faas)

* Acknowledgements: 

  This post is based upon [Deploy your Serverless Python function locally with OpenFaas in Kubernetes](https://dev.to/yankee/deploy-your-serverless-python-function-locally-with-openfaas-in-kubernetes-18jf) by Yankee Maharjan.