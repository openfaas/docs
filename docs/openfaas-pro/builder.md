# Function Builder API

The Function Builder API provides a simple REST API to create your functions from source code.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

This is ideal if:

* You're offering OpenFaaS as a service provider

  You can invoke the Function Builder's API from your own product
* You need to build dozens or hundreds of functions

  Do you really want to manage and maintain that many CI jobs in Jenkins or GitLab?
* You're already building images in-cluster with the Docker Socket

  You realise how bad this is, and you want a more secure alternative
  
The Pro Builder uses Buildkit, developed by the Docker community to perform fast, cached, in-cluster builds via a HTTP API and uses mTLS for encryption.

Various self-hosted, open source and managed registries are supported.

## Installation

The Pro Builder is available to OpenFaaS Pro customers.

Install the builder using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder).

## Usage

Create a build context using the `faas-cli build --shrinkwrap` command:

```bash
# Prepare a temporary directory
rm -rf /tmp/functions
mkdir -p /tmp/functions
cd /tmp/functions

# Create a new function
faas-cli new --lang node16 hello-world

# The shrinkwrap command performs the templating 
# stage, then stops before running "docker build"

# Look in ./build/hello-world to see the contents 
# that is normally passed to "docker build"
faas-cli build --shrinkwrap -f hello-world.yml

# Now rename "hello-world" to "context"
# since that's the folder name expected by the builder
cd build
rm -rf context
mv hello-world context

# Create a config file with the registry and the 
# image name that you want to use for publishing the 
# function.
export DOCKER_USER=alexellis2
echo -n '{"image": "docker.io/'$DOCKER_USER'/test-image-hello:0.1.0"}' > com.openfaas.docker.config
```

> As an alternative to a private or authenticated registry, you can use [ttl.sh by Replicated](https://ttl.sh) as a temporary registry for testing (only). It allows you to publish containers that are removed after a certain time-limit, try `ttl.sh/test-image-hello:1h` for an image that is removed after 1 hour.

If you wish, you can also construct this filesystem using your own application, but it is easier to execute the `faas-cli` command from your own code.

Then create a tar archive of the context of the `/tmp/functions/build/` directory:

```bash
tar cvf req.tar  --exclude=req.tar  .
```

The format will be as follows:

```bash
./com.openfaas.docker.config
./context/
./context/index.js
./context/template.yml
./context/Dockerfile
./context/function/
./context/function/package.json
./context/function/handler.js
./context/package.json
./context/.dockerignore
```

Now port-forward the service and invoke it:

```bash
kubectl port-forward -n openfaas \
    deploy/pro-builder 8081:8080
```

Generate a SHA256 HMAC signature and invoke the function passing in the `X-Build-Signature` header.

Invoke a build:

```bash
PAYLOAD=$(kubectl get secret -n openfaas payload-secret -o jsonpath='{.data.payload-secret}' | base64 --decode)

HMAC=$(cat req.tar | openssl dgst -sha256 -hmac $PAYLOAD | sed -e 's/^.* //')

curl -H "X-Build-Signature: sha256=$HMAC" -s http://127.0.0.1:8081/build -X POST --data-binary @req.tar | jq

....
    "v: 2021-10-20T16:48:34Z exporting to image 8.01s"
  ],
  "imageName": "docker.io/alexellis2/test-image-hello:0.1.0",
  "status": "success"
}
```

## In-cluster access

You can access the Pro Builder via your code, or an OpenFaaS function by specifying its URL and its in-cluster service name. You do not need to expose the builder's API publicly.

```
http://pro-builder.openfaas:8080/build
```

## Build arguments

You may need to enable build arguments for the Dockerfile, these can be passed through the configuration file.

```json
{
  "image": "docker.io/alexellis2/test-image:0.1.0",
  "buildArgs": {
    "BASE_IMAGE": "gcr.io/quiet-mechanic-140114/openfaas-base/node16"
  }
}
```

## Scaling the builder

The Pro Builder can be scaled out, which also deploys additional replicas of Buildkit:

```bash
kubectl scale -n openfaas deploy/pro-builder \
  --replicas=3
```

## Monitoring the builder

The builder has additional metrics which will be scraped by the Prometheus instance installed by the OpenFaaS helm chart.

![Metrics for the builder](/images/builder-metrics.png)

> Pictured: metrics for the builder showing inflight builds, average duration and HTTP status codes to detect errors.

* [Download the Grafana JSON file from the Customer Community](https://github.com/openfaas/openfaas-pro/tree/master/dashboards)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
