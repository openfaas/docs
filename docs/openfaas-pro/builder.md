# Function Builder API

The Function Builder API provides a simple REST API to create your functions from source code.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

The Function Builder is designed to be integrated via HTTP to automate building images in your cluster, and for service providers.

## So is it right for you? 

* You're a service provider with custom functionality or functions

    If you offer a way for customers to provide you custom code, you can invoke the Function Builder API to create a container image, which you can then deploy via the OpenFaaS REST API.

    This means you can extend your platform for customers with a few simple steps.

* You manage dozens or hundreds of functions

    Instead of defining hundreds of different CI jobs or definitions in Jenkins, GitHub Actions, or GitLab, you can integrate with the Builder's REST API to build functions programmatically.

    That means you have less to maintain and keep up to date, particularly if you need to make a change across your functions later down the line or if you want to apply policies and governance.

* You're already building images in-cluster

    If you're sharing a Docker socket from the host into your cluster, or running a container with Docker in Docker (DIND), or in privileged mode, this is making your cluster vulnerable to serious attacks.

    The Function Builder API builds images in-cluster, but can run without root privileges, or needing to run Docker.

The Function Builder uses Buildkit, developed by the Docker community to perform fast, cached, in-cluster builds via a HTTP API and uses mTLS for encryption.

You can use various self-hosted, open-source and managed cloud container registries with the Function Builder API.

## Installation

The Function Builder is available to OpenFaaS Pro customers.

Install the builder using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder).

See also: [code samples with Node.js, Python, Go and PHP](https://github.com/openfaas/function-builder-examples)

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
  "image": "docker.io/alexellis2/test-image-hello:0.1.0",
  "status": "success"
}
```

## HTTP client examples

A HTTP client has three tasks to perform:

1. Construct a folder called `context` with the files needed to build a container

    `faas-cli build --shrinkwrap` can help here, and allow you to use the existing templates we provide, or one of your own.

    Any valid folder with a Dockerfile will work.

3. Create a configuration file

    The configuration file should be called `com.openfaas.docker.config` and be placed outside of the `context` folder (see the example above with curl)

3. Create a tar file

    Create a tar file which contains `com.openfaas.docker.config` and `context/*`.

4. Calculate the HMAC of the tar file

    Calculate the HMAC of the tar file using a standard crypto library, you'll also need to input the payload secret for the function builder.

5. Invoke the API via HTTP

    Next, invoke the API's `/build` endpoint.

    You'll receive a JSON result with the status, logs and an image name if the image was published successfully.

    ```json
    {
        "log": [
            "v: 2022-06-23T09:10:12Z [ship 15/16] RUN npm test 0.35s",
            "v: 2022-06-23T09:10:13Z [ship 16/16] WORKDIR /home/app/",
            "v: 2022-06-23T09:10:13Z [ship 16/16] WORKDIR /home/app/ 0.09s",
            "v: 2022-06-23T09:10:13Z exporting to image",
            "s: 2022-06-23T09:11:06Z pushing manifest for ttl.sh/openfaas-image:1h@sha256:b077f553245c09d789980d081d33d46b93a23c24a5ec0a9c3c26be2c768db93e 0",
            "s: 2022-06-23T09:11:09Z pushing manifest for ttl.sh/openfaas-image:1h@sha256:b077f553245c09d789980d081d33d46b93a23c24a5ec0a9c3c26be2c768db93e 0",
            "v: 2022-06-23T09:10:13Z exporting to image 5.18s"
        ],
        "image": "ttl.sh/openfaas-image:1h",
        "status": "success"
    }
    ```

There are several examples available of how to call the Function Builder's API via different programming languages: [openfaas-function-builder-api-examples](https://github.com/openfaas/function-builder-examples)

You should be able to translate the example given with curl into any programming language, but if you need additional support, feel free to [reach out to us](https://openfaas.com/support).

## Monitoring the builder

The builder has additional metrics which will be scraped by the Prometheus instance installed by the OpenFaaS helm chart.

![Metrics for the builder](/images/builder-metrics.png)

> Pictured: metrics for the builder showing inflight builds, average duration and HTTP status codes to detect errors.

* [Download the Grafana JSON file from the Customer Community](https://github.com/openfaas/openfaas-pro/tree/master/dashboards)

## In-cluster access

You can access the Function Builder via your code, or an OpenFaaS function by specifying its URL and its in-cluster service name. You do not need to expose the builder's API publicly.

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

The Function Builder can be scaled out, which also deploys additional replicas of Buildkit:

```bash
kubectl scale -n openfaas deploy/pro-builder \
  --replicas=3
```

## Limiting the amount of concurrent requests

You can limit the amount of concurrent requests that a builder will accept by setting `proBuilder.maxInflight: N` within the helm chart or the `max_inflight` environment variable on the Deployment.

Once in place, a busy worker will return responses like this:

```bash
HTTP/1.1 429 Too Many Requests
Date: Tue, 27 Sep 2022 11:07:06 GMT
Content-Length: 62
Content-Type: text/plain; charset=utf-8
Connection: close

Concurrent request limit exceeded. Max concurrent requests: 1
```

The pro-builder will be marked as unready by Kubernetes, and if you have other replicas (see above section), then when you retry the request, it should hit a ready worker instead.

**Why not use a function to invoke the API?**

If you do not have code to retry invocations in your own product/system, OpenFaaS Pro supports this through its async queue-worker, and you could use a function in a separate namespace like `openfaas-system` to queue builds more reliably during busy periods. Feel free to reach out to us if you have questions about this approach. 

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
