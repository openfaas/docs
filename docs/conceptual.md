# Conceptual design of OpenFaaS

![Layers](https://github.com/openfaas/faas/raw/master/docs/of-layer-overview.png)

### Function Watchdog

* You can make any Docker image into a serverless function by adding the *Function Watchdog* (a tiny Golang HTTP server)
* The *Function Watchdog* is the entrypoint allowing HTTP requests to be forwarded to the target process via STDIN. The response is sent back to the caller by writing to STDOUT from your application.

### API Gateway / UI Portal

* The API Gateway provides an external route into your functions and collects Cloud Native metrics through Prometheus.
* Your API Gateway will scale functions according to demand by altering the service replica count in the Docker Swarm or Kubernetes API.
* A UI is baked in allowing you to invoke functions in your browser and create new ones as needed.

> The API Gateway is a RESTful micro-service and you can view the [Swagger docs here](/architecture/gateway/#swagger).

### CLI

Any container or process in a Docker container can be a serverless function in FaaS. Using the [FaaS CLI](http://github.com/openfaas/faas-cli) you can deploy your functions quickly.

Create new functions from templates for Node.js, Python, [Go](https://blog.alexellis.io/serverless-golang-with-openfaas/) and many more. If you can't find a suitable template you can also use a Dockerfile.

> The CLI is effectively a RESTful client for the API Gateway.

When you have OpenFaaS configured you can [get started with the CLI here](https://blog.alexellis.io/quickstart-openfaas-cli/)

### Function examples

You can generate new functions using the FaaS-CLI and built-in templates or use any binary for Windows or Linux in a Docker container.

* Python example:

```python
import requests

def handle(req):
    r =  requests.get(req, timeout = 1)
    print(req +" => " + str(r.status_code))
```
*handler.py*

* Node.js example:

```js
"use strict"

module.exports = (callback, context) => {
    callback(null, {"message": "You said: " + context})
}
```
*handler.js*

Other [Sample functions](https://github.com/openfaas/faas/tree/master/sample-functions) are available in the Github repository in a range of programming languages.

## Documentation

[View our guides](https://github.com/openfaas/faas/tree/master/guide) for documentation, deployment guides and tutorials.
