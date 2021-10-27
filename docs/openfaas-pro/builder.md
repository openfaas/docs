# Function Builder

OpenFaaS Pro offers an API called the Pro Builder which you can run in your cluster to build container images from functions.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

There are many tools that you can use to build OpenFaaS container images, so why would you need the Pro Builder?

* You want to build images via API
* You need to build many images and other tooling makes this hard to scale
* You want to build images within your cluster, rather than using separate external infrastructure
* You want a more secure method than mounting a Docker socket into your container

The Pro Builder uses Buildkit, developed by the Docker community to perform fast, cached, in-cluster builds via a HTTP API and uses mTLS for encryption.

Various self-hosted, open source and managed registries are supported.

## Installation

The Pro Builder is available to OpenFaaS Pro customers. Email your support contact for more information.

## Usage

Create a build context using the `faas-cli build --shrinkwrap` command:

```bash
rm -rf /tmp/functions
mkdir -p /tmp/functions

faas-cli new --lang node14 hello-world
faas-cli build --shrinkwrap -f hello-world.yml

cd build
rm -rf context
mv hello-world context

export DOCKER_USER=alexellis2
echo -n '{"image": "docker.io/'$DOCKER_USER'/test-image-hello:0.1.0"}' > com.openfaas.docker.config
```

If you wish, you can also construct this filesystem using your own application, but it is easier to execute the faas-cli command from your own code.

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

Passing in additional build-args:

```json
{
  "image": "docker.io/alexellis2/test-image:0.1.0",
  "buildArgs": {
    "BASE_IMAGE": "gcr.io/quiet-mechanic-140114/openfaas-base/node14"
  }
}
```

You can access the Pro Builder via your code, or an OpenFaaS function by specifying its URL and its in-cluster service name:

```
http://pro-builder.openfaas:8080/build
```

The Pro Builder can be scaled out, which also deploys additional replicas of Buildkit:

```bash
kubectl scale -n openfaas deploy/pro-builder \
  --replicas=3
```
