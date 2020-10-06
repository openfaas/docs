# TestDrive

## TestDrive (v2)

The TestDrive has been replaced by the OpenFaaS workshop which is a set of self-paced labs developed by the community to teach you the basics of building Serverless Functions with OpenFaaS.

Please start [Lab 1 here](https://github.com/openfaas/workshop)

## TestDrive (v1 - classic version)

OpenFaaS (or Functions as a Service) is a framework for building serverless functions on Docker Swarm and Kubernetes with first class metrics. Any UNIX process can be packaged as a function in FaaS enabling you to consume a range of web events without repetitive boiler-plate coding.

> Please support the project and put a **Star** on the repo.

OpenFaaS is Kubernetes and Docker-native. In this test-drive we'll be using Docker Inc's online lab for Docker Swarm called play-with-docker.

# Overview

You'll be up and running in a few minutes and invoking functions via the UI, the CLI or via `curl`. You can deploy one of the ready-made functions from the built-in Function Store and try out everything from figlet, to OCR, to colorisation with machine-learning.

When you're ready to build your own function head over to the following tutorial:

* [Morning coffee with the OpenFaaS CLI](https://blog.alexellis.io/quickstart-openfaas-cli/)

## Pre-reqs

The guide makes use of a cloud playground service called [Play with Docker](https://labs.play-with-docker.com/) that provides free Docker hosts that expire after around 5 hours. Once you are familiar with the workflow, you can also deploy to your own laptop or cloud host.

## Start here

Head over to [https://labs.play-with-docker.com/](https://labs.play-with-docker.com/) and start a new session. You may have to have to fill out a Captcha and log in with your [Docker Hub account](https://hub.docker.com).

* Click "Add New Instance" to create a single Docker host (more can be added later)

This one-shot script clones the code, sets up a Docker Swarm master node then deploys OpenFaaS with the sample stack:

```
# docker swarm init --advertise-addr eth0 && \
  git clone https://github.com/openfaas/faas && \
  cd faas && \
  ./deploy_stack.sh --no-auth
```

*The shell script makes use of a v3 docker-compose.yml file - read the `deploy_stack.sh` file for more details.*

> If you are not testing on play-with-docker then remove `--advertise-addr eth0` from first line of the script.

* Now that everything's deployed take note of the two ports at the top of the screen:

* 8080 - the API Gateway and OpenFaaS UI
* 9090 - the Prometheus metrics endpoint

![](https://user-images.githubusercontent.com/6358735/31058899-b34f2108-a6f3-11e7-853c-6669ffacd320.jpg)

We passed a flag of `--no-auth` to disable authentication, you can leave this off to enable authentication for the OpenFaaS gateway.

## Install FaaS-CLI

We will also install the OpenFaaS CLI which can be used to create, list, invoke and remove functions. You can do this on the play-with-docker (PWD) manager directly, or on your laptop. If you install this on your laptop then make sure you pass the `--gateway` flag with the address provided on PWD.

```shell
$ curl -sL cli.openfaas.com | sh
```

On your own machine change ` | sh` to ` | sudo sh`, for MacOS you can just use `brew install faas-cli`.

* Find out what you can do

```shell
$ faas-cli --help
```

## Deploy the classic sample functions

OpenFaaS functions can be deployed from a YAML file called a stack file. Deploy the classic sample functions by running the following command:

```
faas-cli deploy -f stack.yml
```

If your stack file is called `stack.yml`, then you can leave off the `-f` parameter. The flag `-f` or `--yaml` can also accept a HTTP(s) address to deploy straight from GitHub raw or similar.

Some of the functions in the stack include:

* Markdown to HTML renderer (markdownrender) - takes .MD input and produces HTML (Golang)
* Docker Hub Stats function (hubstats) - queries the count of images for a user on the Docker Hub (Golang)
* Node Info (nodeinfo) function - gives you the OS architecture and detailled info about the CPUS (Node.js)
* Webhook stasher function (webhookstash) - saves webhook body into container's filesystem - even binaries (Golang)

### Invoke the sample functions

You can access functions via the command line using `curl` with a HTTP request, by using the `faas-cli`, with the built-in UI or even [Postman](https://www.getpostman.com).

* Invoke the markdown render function with the CLI:

```
$ echo "# Test *Drive*" | faas-cli invoke markdown
<h1>Test <em>Drive</em></h1>
```

You can also type in multiple lines followed by Control + D:
```
$ faas-cli invoke markdown
# Line 1
## Line 2

<h1>Line 1</h1>
<h2>Line 2</h2>
```

* List your functions

```
$ faas-cli list
Function                        Invocations     Replicas
echoit                     0               1
base64                     0               1
decodebase64               0               1
markdown                   3               1
nodeinfo                   0               1
wordcount                  0               1
hubstats                   0               1
webhookstash               0               1
```

You can also pass the flag `-v` to see the Docker images being used.

* Use `curl` or HTTP

If you get `$OPENFAAS_URL` to the URL for your gateway the following will work:

```
export OPENFAAS_URL="http://..."

curl -d "# Line1" -i http://$OPENFAAS_URL/function/markdown
```

* UI portal:

The UI portal is accessible on: http://127.0.0.1:8080/ - it show a list of functions deployed on your swarm and allows you to test them out.

View screenshot:

<a href="https://pbs.twimg.com/media/C3hDUkyWEAEgciP.jpg"><img src="https://pbs.twimg.com/media/C3hDUkyWEAEgciP.jpg" width="800"></img></a>

You can find out which services are deployed like this:

```
# docker stack ls
NAME  SERVICES
func  3

# docker stack ps func
ID            NAME               IMAGE                                  NODE  DESIRED STATE  CURRENT STATE         
rhzej73haufd  func_gateway.1     alexellis2/faas-gateway:latest         moby  Running        Running 26 minutes ago
fssz6unq3e74  hubstats.1    alexellis2/faas-dockerhubstats:latest  moby  Running        Running 27 minutes ago
nnlzo6u3pilg  func_prometheus.1  quay.io/prometheus/prometheus:latest   moby  Running        Running 27 minutes ago
```

* Head over to http://127.0.0.1:9090 for your Prometheus metrics
 * A saved Prometheus view is available here: [metrics overview](http://127.0.0.1:9090/graph?g0.range_input=15m&g0.expr=rate(gateway_function_invocation_total%5B20s%5D)&g0.tab=0&g1.range_input=15m&g1.expr=gateway_functions_seconds_sum+%2F+gateway_functions_seconds_counts&g1.tab=0&g2.range_input=15m&g2.expr=gateway_service_count&g2.tab=0)

## Build functions from templates and the CLI

The following guides show how to use the CLI and code templates to build functions.

Using a template means you only have to write a handler file in your chosen programming language. To see a list of the official language templates:

```
faas-cli template pull

faas-cli new --list

Languages available as templates:
- csharp
- dockerfile
- go
- java8
- node
- python
- python3
- ruby
```

> PHP and other languages are available by running `faas-cli template pull` with various Git repos maintained by the wider OpenFaaS community.

For anything else you can build your own template or use the `dockerfile` type to use an existing containerized microservice as your function.

Guides:

* [Your first serverless Python function with OpenFaaS](https://blog.alexellis.io/first-faas-python-function/)

* [Your first serverless .NET / C# function with OpenFaaS](https://medium.com/@rorpage/your-first-serverless-net-function-with-openfaas-27573017dedb)

## Package a custom Docker image

Read the developer guide:

* [Packaging a function](https://github.com/openfaas/faas/blob/master/DEV.md)

The original blog post also walks through creating a function:

* [FaaS blog post](http://blog.alexellis.io/functions-as-a-service/)

## Add new functions to FaaS at runtime

**Option 1: via the FaaS CLI**

The FaaS CLI can be used to build functions very quickly though the use of templates. See more details on the FaaS CLI [here](https://github.com/openfaas/faas-cli).

**Option 2: via FaaS UI portal**

To attach a function at runtime you can use the "Create New Function" button on the portal UI at http://127.0.0.1:8080/ 

<a href="https://pbs.twimg.com/media/C8opW3RW0AAc9Th.jpg:large"><img src="https://pbs.twimg.com/media/C8opW3RW0AAc9Th.jpg:large" width="600"></img></a>

Creating a function via the UI:

| Option                 | Usage             |
|------------------------|--------------|
| `Image`		 	| The name of the image you want to use for the function. A good starting point is functions/alpine |
| `Service Name`  	 	| Describe the name of your service. The Service Name format is: [a-zA-Z_0-9] |
| `fProcess` 		 	| The process to invoke for each function call. This must be a UNIX binary and accept input via STDIN and output via STDOUT. |
| `Network`		 	| The network `func_functions` is the default network. |

Once the create button is clicked, faas will provision a new Docker Swarm service. The newly created function will shortly be available in the list of functions on the left hand side of the UI.

**Option 3: Programatically through a HTTP POST to the API Gateway**

A HTTP post can also be sent via `curl` etc to the endpoint used by the UI (HTTP post to `/system/functions`)

```go
// CreateFunctionRequest create a function in the swarm.
type CreateFunctionRequest struct {
	Service    string `json:"service"`
	Image      string `json:"image"`
	Network    string `json:"network"`
	EnvProcess string `json:"envProcess"`
}
```

Check the [Swagger API](https://raw.githubusercontent.com/openfaas/faas/master/api-docs/swagger.yml) for more details of additional fields such as `Labels` and `Constraints`.

Example:

For a quote-of-the-day type of application:

```
curl 127.0.0.1:8080/system/functions -d '
{"service": "oblique", "image": "vielmetti/faas-oblique", "envProcess": "/usr/bin/oblique", "network": "func_functions"}'
```

For a hashing algorithm:

```
curl 127.0.0.1:8080/system/functions -d '
{"service": "stronghash", "image": "functions/alpine", "envProcess": "sha512sum", "network": "func_functions"}'
```

### Delete a function at runtime

You can delete a function through the FaaS-CLI or with the Docker CLI

```
$ faas-cli remove echoit
```

### Exploring the functions with `curl`

**Sample function: Docker Hub Stats (hubstats)**

```
# curl -X POST http://127.0.0.1:8080/function/hubstats -d "alexellis2"
The organisation or user alexellis2 has 99 repositories on the Docker hub.
```

The `-d` value passes in the argument for your function. This is read via STDIN and used to query the Docker Hub to see how many images you've created/pushed.

You can also invoke functions using the OpenFaaS CLI:

```
# echo -n "library" | faas-cli invoke hubstats
The organisation or user library has 128 repositories on the Docker hub.
```

**Sample function: Node OS Info (nodeinfo)**

Grab OS, CPU and other info via a Node.js container using the `os` module.

If you invoke this method in a while loop or with a load-generator tool then it will auto-scale to 5, 10, 15 and finally 20 replicas due to the load. You will then be able to see the various Docker containers responding with a different Hostname for each request as the work is distributed evenly.

Here is a loop that can be used to invoke the function in a loop to trigger auto-scaling.
```
while [ true ] ; do curl -X POST http://127.0.0.1:8080/function/nodeinfo -d ''; done
```

Example:

```
# curl -X POST http://127.0.0.1:8080/function/nodeinfo -d ''

Hostname: 9b077a81a489

Platform: linux
Arch: arm
CPU count: 1
Uptime: 776839
```

To control scaling behaviour you can set a min/max scale value with a label when deploying your function via the CLI or the API:

```
  labels:
    "com.openfaas.scale.min": "5"
    "com.openfaas.scale.max": "15"
```

**Sample function: webhook stasher (webhookstash)**

Another cool sample function is the Webhook Stasher which saves the body of any data posted to the service to the container's filesystem. Each file is written with the filename of the UNIX time.

```
# curl -X POST http://127.0.0.1:8080/function/webhookstash -d '{"event": "fork", "repo": "alexellis2/faas"}'
Webhook stashed

# docker ps|grep stash
d769ca70729d        alexellis2/faas-webhookstash@sha256:b378f1a144202baa8fb008f2c8896f5a8

# docker exec d769ca70729d find .
.
./1483999546817280727.txt
./1483999702130689245.txt
./1483999702533419188.txt
./1483999702978454621.txt
./1483999703284879767.txt
./1483999719981227578.txt
./1483999720296180414.txt
./1483999720666705381.txt
./1483999720961054638.txt
```

> Why not start the code on play-with-docker.com and then configure a Github repository to send webhooks to the API Gateway?

### Wrapping up

Please show your support for Open Source and head over to the [Github repo and Star the project](https://github.com/openfaas/faas).
