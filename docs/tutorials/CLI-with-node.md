# Tutorial for the OpenFaaS CLI with Node.js

We'll explore how to create, build, deploy and invoke a brand new function with one of the supported language templates. We'll use the CLI for every part of the workflow.

**What is OpenFaas?**

![](https://blog.alexellis.io/content/images/2017/08/small.png)

OpenFaaS is a framework for packaging code, binaries or containers as Serverless functions on any platform - Windows or Linux. [Visit website](https://www.openfaas.com/)

In the next few minutes we'll:

* Create a function from a code template
* Build the function as a Docker image
* Push the image to a Docker registry
* Deploy the function
* Invoke the function

..and more

### Pre-requisites

Before starting you should setup OpenFaaS on your laptop or cluster using a deployment guide:

[Deployment guide](http://docs.openfaas.com/deployment/)

### Get the CLI

For the latest version of the CLI type in the following:

```
$ curl -sL https://cli.openfaas.com | sudo sh
```

> This is also available via `brew install faas-cli` on MacOS.

### Find the help page

All the commands have a `--help` flag available which provides documentation on usage:

```
$ faas-cli --help

Manage your OpenFaaS functions from the command line

Usage:
  faas-cli [flags]
  faas-cli [command]

Available Commands:
  build          Builds OpenFaaS function containers
  deploy         Deploy OpenFaaS functions
  help           Help about any command
  push           Push OpenFaaS functions to remote registry (Docker Hub)
  remove         Remove deployed OpenFaaS functions
  version        Display the clients version information

Flags:
  -h, --help          help for faas-cli
  -f, --yaml string   Path to YAML file describing function(s)

Use "faas-cli [command] --help" for more information about a command.
```

### Create a new Node.js function

We'll create a brand new Node.js template and a YAML file at the same time which is used by the CLI to save you on typing.

```
$ faas-cli new callme --lang node
Folder: callme created.
Function created in folder: callme
Stack file written: callme.yml
```

You'll see the following was created:

```
$ find . | grep callme
./callme
./callme/handler.js
./callme/package.json
./callme.yml
```

Here's the YAML file which was generated:

```
$ cat callme.yml
provider:
  name: openfaas
  gateway: http://localhost:8080

functions:
  callme:
    lang: node
    handler: ./callme
    image: callme
```

The contents of `callme.yml` can now be used with the CLI to save on typing and build, push, deploy and invoke your function.

> If your cluster is remote or not running on port 8080 - then edit this in the YAML file before continuing.

A handler.js file was generated for your function which looks like this:

```
"use strict"

module.exports = (context, callback) => {
    callback(undefined, {status: "done"});
}
```

It will just send back a status of "done" when called.

> Tip: You can also create your `handler.js` file manually

*What about `npm` modules etc?*

You can edit the `package.json` file and your dependencies will be installed during the "build" step. The same works for the other language templates with a `requirements.txt` file for Python etc.

### Build the function

The local Docker client is used to build your function into a Docker image.

```
$ faas-cli build -f callme.yml 
Building: callme.
Clearing temporary build folder: ./build/callme/
Preparing ./callme/ ./build/callme/function
Building: callme with node template. Please wait..
docker build -t callme .
Sending build context to Docker daemon  8.704kB
Step 1/19 : FROM node:6.11.2-alpine
 ---> 16566b7ed19e
...

Step 19/19 : CMD fwatchdog
 ---> Running in 53d04c1631aa
 ---> f5e1266b0d32
Removing intermediate container 53d04c1631aa
Successfully built f5e1266b0d32
Successfully tagged callme:latest
Image: callme built.
```

### Push your function

It is recommended to change the "tag" in the image field after each change, but by default OpenFaaS will attempt to pull your function from the Docker Hub or a remote container registry. This enables you to iterate on your functions quickly without changing their Docker "tag".

> If you are using a single-node Docker Swarm cluster on your laptop then you can skip this step. If you are using `minikube` or a single-node Kubernetes cluster then you can set the ImagePullPolicy to "IfNotPresent" in the helm chart for the Kubernetes controller, this will allow you to work with images within your library.

Now edit the `callme.yml` YAML file and set the "image" line to your username on the Docker Hub such as: `alexellis/callme`. Then build the function again.

```
$ faas-cli push -f callme.yml 
Pushing: callme to remote repository.
The push refers to a repository [docker.io/alexellis/callme]
...
```

Once the image is pushed up to the Docker Hub or another remote Docker registry we can deploy and run the function.

### Deploy the function

```
$ faas-cli deploy -f callme.yml
Deploying: callme.
No existing service to remove
Deployed.
200 OK
URL: http://localhost:8080/function/callme
```

If an existing / old function was already deployed then it will be removed first.

> If you get an error at this point then please make sure you followed the pre-requisites. 

### Invoke the function

We can now invoke the function:

```
$ faas-cli invoke -f callme.yml callme
Reading from STDIN - hit (Control + D) to stop.
This is my message

{"status":"done"}
```

You can also pipe a command into the function.

```
$ date | faas-cli invoke -f callme.yml callme
{"status":"done"}
```

**Task:** Can you edit the function so that it returns the input along with "status": "done"? Just edit the code, run "build", "push" and "deploy" - then  you're good to invoke it again.

### List the functions

You can list your functions and find out how many replicas they have and also the invocation count like this:

```
$ faas-cli list
Function                      	Invocations    	Replicas
inception                     	0              	1    
callme                        	2              	1    
func_hubstats                 	0              	1    
tensorflow                    	0              	1    
func_echoit                   	0              	1    
func_nodeinfo                 	0              	1   
```

> Tip: If you pass `--verbose` then you'll also see the image name

### Remove the function

You can remove functions by using the CLI. If you specify a YAML file the CLI will remove all the deployed functions listed in the file.

```
$ faas-cli rm -f callme.yml 
Deleting: callme.
Removing old service.
```

Type in `faas-cli rm --help` for more information.

## Wrapping up

> Did you know we also have a UI and Prometheus metrics built-into the stack? Head over to the [GitHub repo](https://github.com/alexellis/faas) to read more and to **Star** the repository.

That concludes the coffee break - we just built and deployed our first Serverless Node.js OpenFaaS function - "Look Ma! No UI!" You can find help for any of the commands by passing in the `--help` parameter.

* Acknowledgements:

This post is based upon [Morning coffee with the OpenFaaS CLI](https://blog.alexellis.io/quickstart-openfaas-cli/) by Alex Ellis
