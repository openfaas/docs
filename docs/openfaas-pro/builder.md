# Function Builder API

The Function Builder API provides a simple REST API to create your functions from source code.

!!! info "OpenFaaS Enterprise feature"
    This feature is included for [OpenFaaS Enterprise](/openfaas-pro/introduction) customers.

The Function Builder is designed to be integrated via HTTP to automate building images in your cluster, and for service providers.

> See also: [How to build functions from source code with the Function Builder API](https://www.openfaas.com/blog/how-to-build-via-api/)

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

* You are using GovCloud

    If you're using GovCloud, then your auditer or compliance partner may have already told you that your product should not have external dependencies on separate build systems. The Function Builder API can be run rootless, without privileges and without needing to share a Docker socket.

The Function Builder uses Buildkit, developed by the Docker community to perform fast, cached, in-cluster builds via a HTTP API and uses mTLS for encryption.

You can use various self-hosted, open-source and managed cloud container registries with the Function Builder API.

## Installation

The Function Builder is available to OpenFaaS Pro customers.

Install the builder using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder).

See also: [code samples with Node.js, Python, Go and PHP](https://github.com/openfaas/function-builder-examples)

## Usage

We provide the following examples to help you explore and get started:

* Remote builds via `faas-cli` for testing purposes
* Step by step commands with `curl` and bash to show you how the workflow works
* Code examples in various languages to show you how to integrate for production

!!! note
    Authentication is required to publish images to remote registries. You must make sure that you do not include a configuration file from your machine directly, if the credentials keystore is enabled in Docker Desktop. See the [helm chart for more details](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder).

### Remote builds with `faas-cli`

The `faas-cli` `publish` and `up` commands can use the `--remote-builder` flag to perform remote builds with the Function Builder. The path to the file containing the payload secret should be specified through the `--payload-secret` flag.

First, port-forward the builder, and obtain the payload secret:

```
kubectl port-forward -n openfaas \
    deploy/pro-builder 8081:8080
export PAYLOAD=$(kubectl get secret -n openfaas payload-secret -o jsonpath='{.data.payload-secret}' | base64 --decode)
echo $PAYLOAD > $HOME/.openfaas/payload.txt
```

Create a test function using the `python3` template, and set it to publish to `ttl.sh`, an ephemeral registry that doesn't require authentication:

```bash
faas-cli new --prefix ttl.sh/test-images \
    --lang python3 py-fn
mv py-fn.yml stack.yml
```

Now, publish an image using the remote builder:

```bash
faas-cli publish --remote-builder http://127.0.0.1:8081/build \
  --payload-secret $HOME/.openfaas/payload.txt
```

To deploy the image that you've just built:

```bash
faas-cli deploy
```

Or to publish _and_ then deploy that image to the cluster in one go:

```
faas-cli up --remote-builder http://127.0.0.1:8081/build \
  --payload-secret $HOME/.openfaas/payload.txt
```

### Remote builds via `curl`

As an alternative to a private or authenticated registry, you can use [ttl.sh by Replicated](https://ttl.sh) as a temporary registry for testing (only). It allows you to publish containers that are removed after a certain time-limit, try `ttl.sh/test-image-hello:1h` for an image that is removed after 1 hour.

You can use any registry that is configured in your `config.json` file by changing the `image` field in the configuration file.

Create a build context using the `faas-cli build --shrinkwrap` command:

```bash
# Prepare a temporary directory
rm -rf /tmp/functions
mkdir -p /tmp/functions
cd /tmp/functions

# Create a new function
faas-cli new --lang node20 hello-world

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
export REGISTRY=ttl.sh
export OWNER=alexellis2
echo -n '{"image": "'$REGISTRY'/'$OWNER'/test-image-hello:0.1.0", "platforms": ["linux/amd64","linux/arm64"]}' > com.openfaas.docker.config
```

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
  "image": "ttl.sh/alexellis2/test-image-hello:0.1.0",
  "status": "success"
}
```

The initial build is likely to take some time, however, if you run the build again or only change some text within a file the subsequent build could complete with single-digit seconds.

### Remote builds with a HTTP client

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

## How to monitor the builder

The builder has additional metrics which will be scraped by the Prometheus instance installed by the OpenFaaS helm chart.

![Metrics for the builder](/images/builder-metrics.png)

> Pictured: metrics for the builder showing inflight builds, average duration and HTTP status codes to detect errors.

* [Download the Grafana JSON file from the Customer Community](https://github.com/openfaas/customers/tree/master/dashboards)

## How to gain in-cluster access

You can access the Function Builder via your code, or an OpenFaaS function by specifying its URL and its in-cluster service name.

```
http://pro-builder.openfaas:8080/build
```

Even though the endpoint is authenticated, we recommend you do not expose the builder's API to the Internet.

## How to specify build arguments

You may need to enable build arguments for the Dockerfile, these can be passed through the configuration file.

```json
{
  "image": "ttl.sh/alexellis/test-image:0.1.0",
  "buildArgs": {
    "BASE_IMAGE": "gcr.io/quiet-mechanic-140114/openfaas-base/node16"
  }
}
```

## How to perform multi-arch builds

You may wish to cross-compile a function to run on an arm64 host, if so, you can provide a `platform` key in the configuration file.

The below will build an image for arm64 only and must be deployed only to an arm64 host using OpenFaaS Profiles to ensure it is scheduled correctly to an arm64 host.

```json
{
  "image": "ttl.sh/alexellis/test-image:0.1.0",
  "platforms": ["linux/arm64"]
}
```

Multiple platforms can be specified, the below will build an image for arm64 and amd64, and can be deployed to either type of host.

```json
{
  "image": "ttl.sh/alexellis/test-image:0.1.0",
  "platforms": ["linux/arm64", "linux/amd64"]
}
```

Bear in mind:

* multi-arch images will usually take longer to publish than single-arch images due to emulation with QEMU
* any steps performed under a TARGETARCH which differs from BUILDARCH will be emulated with QEMU which will add overhead to the build process - you can mitigate this by running a dedicated arm64 and amd64 pro-builder Helm chart installation

## How to scale the builder

The Function Builder can be scaled out, which also deploys additional replicas of the Function Builder:

```bash
kubectl scale -n openfaas deploy/pro-builder \
  --replicas=3
```

## How to limiting the amount of concurrent builds

You can limit the amount of concurrent requests that a builder will accept by setting `proBuilder.maxInflight: N` within the helm chart or the `max_inflight` environment variable on the Deployment.

We would encourage you to review the RAM and CPU limits that you've set for your builder, and to experiment with what kind of value to use for `max_inflight`. You could start with a high number, and reduce it over time, until it's too low to get the kind of results you need.

Your total build capacity will be: `max_inflight` * `pro-builder replicas`, so if you have 2 replicas and max_inflight is 3, then you can build 6 functions at once.

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

* [Talk to us](https://openfaas.com/pricing/)
