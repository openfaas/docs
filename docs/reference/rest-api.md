# OpenFaaS REST API

The OpenFaaS REST API along with the Kubernetes CRDs (Function, Profile, JwtIssuer, Policy, Role and Secret) are the primary way to interact with functions.

!!! Info "Learning with the `faas-cli`"

      The `faas-cli` is the original client for the REST API. The source code is available on GitHub at: [openfaas/faas-cli](https://github.com/openfaas/faas-cli).
      One of the easiest ways to develop programs for OpenFaaS is to try out the `faas-cli` commands that you already know, with the `FAAS_DEBUG=1` environment variable set.

      It'll show the HTTP Path, Method and Query String, Body and any additional headers used so that you can build your own integration.

A quick note on TLS: when you expose OpenFaaS over the Internet, you should enable HTTPS to prevent snooping and tampering.

## Documentation

The API documentation is written in OpenAPI and is available on GitHub:

See also: [spec.openapi.yml](https://github.com/openfaas/faas/blob/master/api-docs/spec.openapi.yml)

Note that multiple namespaces are not supported in the Community Edition or Standard Edition of OpenFaaS. There may be additional constraints with the Community Edition, so make sure you review the [comparison table](/openfaas-pro/introduction/).

## SDKs

There is an official Go SDK which is used in the OpenFaaS event connectors and auto-scaler.

* [openfaas/go-sdk](https://github.com/openfaas/go-sdk)

The goals of this SDK are to be lightweight, and intuitive to use, whilst using the same API types that the OpenFaaS gateway and provider use from the [faas-provider's type package](https://github.com/openfaas/faas-provider/tree/master/types).

See also: [Go types used in the OpenFaaS API](https://github.com/openfaas/faas-provider/tree/master/types)

You can generate a client for Node.js, Python, Java and so forth by using the OpenAPI specification available on GitHub, or you could write simple HTTP calls.

The API is designed so that you can write HTTP calls by hand and move very quickly.

## In-cluster vs local access

The OpenFaaS REST API can be accessed from within the cluster or from outside of the cluster.

When accessing the API for development, you may want to port-forward the OpenFaaS gateway to your local machine via `kubectl port-forward`. This talks to the Kubernetes API server so is encrypted.

For any code you deploy within the cluster you should always use `http://gateway.openfaas:8080` as the URL. This will be resolved by the Kubernetes DNS service.

## Authentication

OpenFaaS CE and Standard support Basic Authentication, and OpenFaaS for Enterprises supports IAM using OIDC and JSON Web Tokens (JWT).

### OpenFaaS CE and Standard

Here's an example of how to list functions using the root user account's password:

```bash
export HOST="127.0.0.1:8080"
export PASSWORD=""

curl -s \
    http://admin:$PASSWORD@$HOST/system/functions
```

### OpenFaaS for Enterprises

With OpenFaaS for Enterprises, you can obtain a JWT token for a Kubernetes Service account

[How to authenticate to the OpenFaaS API using Kubernetes JWT tokens](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/)

Or, you can use the `faas-cli` to generate a JWT token:

```bash
faas-cli pro auth --print-token

# Populate from above
TOKEN=""

curl -s \
    -H "Authorization: Bearer $TOKEN"
    http://$HOST/system/functions
```

### Function authentication

For authentication for functions, see [Function Authentication](/reference/authentication/)

## A note on namespaces

In OpenFaaS For Enterprises, all endpoints other than "list namespaces" support a namespace parameter.

Depending on the endpoint, and operation this is passed either through the body or as a query-string parameter.

GET operations tend to supply the namespace in the query-string, whilst POST and PUT operations tend to supply the namespace in the body.

Example with a GET:

```bash
FAAS_DEBUG=1 faas-cli list -n alex
GET http://127.0.0.1:8080/system/functions?namespace=alex
```

Example with a PUT:

```
FAAS_DEBUG=1 faas-cli store deploy -n alex figlet
PUT http://127.0.0.1:8080/system/functions
Content-Type: [application/json]
{"service":"figlet","image":"ghcr.io/openfaas/figlet:latest","namespace":"alex","envProcess":"figlet","labels":{},"annotations":{}}
```

## Functions

### Invocation

Functions can be invoked by adding the `/function/NAME.NAMESPACE` path to the gateway URL.

```bash
curl -s -i http://127.0.0.1:8080/function/env.openfaas.fn
```

If no namespace is specified, then the default namespace is used for the installation, this is usually going to be `openfaas-fn`.

Functions can also be invoked asynchronously by adding the `/async-function/NAME.NAMESPACE` path to the gateway URL. In this case, a HTTP POST is required with the payload in the body.

If a function is being invoked from within Kubernetes, then the following URL should be used:

```
http://gateway.openfaas.svc.cluster.local.:8080
```

Note that in some clusters, the default DNS lookup of "svc.cluster.local" may be different, adapt this as required. The final "." at the end of the line helps to prevent unnecessary DNS lookups.

* [Learn more about async invocations](/reference/async/)

You can expose Functions with a REST-like mapping or on custom subdomains with the `FunctionIngress` Custom Resource:

* See also: [Custom domains and REST-like mappings for functions](/reference/tls-functions)

### Deploy a function

The minimum valid payload includes the function's name and image, however there are many other fields available.

The [function_deployment.go](https://github.com/openfaas/faas-provider/blob/master/types/function_deployment.go) struct from faas-provider provides the full list of fields.

Remember, that `FAAS_DEBUG=1 faas-cli store deploy env` will print out the HTTP request and response for you to learn from.

```
FAAS_DEBUG=1 faas-cli store deploy env

PUT http://127.0.0.1:8080/system/functions

Content-Type: [application/json]
User-Agent: [faas-cli/dev]
Authorization: [Basic REDACTED]

{"service":"env","image":"ghcr.io/openfaas/alpine:latest","envProcess":"env","labels":{},"annotations":{}}
```

### Update a function

To update the previously deployed `env` function with an additional label, run a `PUT` request with the same payload as before, but with the additional label.

```
PASSWORD=""

curl -i \
  -X PUT \
   http://admin:$PASSWORD@127.0.0.1:8080/system/functions \
   -H 'Content-Type: application/json' \
  -d '{"service":"env","image":"ghcr.io/openfaas/alpine:latest","envProcess":"env","labels":{"com.openfaas.scale.zero":"true","com.openfaas.scale.zero-duration":"2m"},"annotations":{}}'
```

Here we've added the label `com.openfaas.scale.zero` which will instruct the OpenFaaS auto-scaler to scale the function to zero replicas after 2 minutes of inactivity.

### Query a function's status

The endpoint to query a function's status returns different data than was inputted via the deploy function endpoint.

Why?

It contains both the deployment data and the status of the function.

```
PASSWORD=""

curl -s \
  -X GET \
   http://admin:$PASSWORD@127.0.0.1:8080/system/function/env?usage=1 \
   -H 'Content-Type: application/json'
```

Adding `?usage=1` includes the RAM and CPU usage of the function for commercial editions of OpenFaaS.

Example output:

```json
{
  "name": "env",
  "image": "ghcr.io/openfaas/alpine:latest",
  "namespace": "openfaas-fn",
  "envProcess": "env",
  "labels": {
    "com.openfaas.scale.zero": "true",
    "com.openfaas.scale.zero-duration": "2m"
  },
  "annotations": {},
  "replicas": 1,
  "availableReplicas": 1,
  "createdAt": "2023-07-04T16:17:24Z",
  "usage": {
    "cpu": 0.28076589450860645,
    "totalMemoryBytes": 2367488
  }
}
```

Note that there is no invocation total on this endpoint, to retrieve that number, you'll need to list all functions.

### List all functions

```
PASSWORD=""

curl -s \
  -X GET \
   http://admin:$PASSWORD@127.0.0.1:8080/system/functions \
   -H 'Content-Type: application/json'
```

Example response:

```json
{
  {
    "name": "env",
    "image": "ghcr.io/openfaas/alpine:latest",
    "namespace": "openfaas-fn",
    "envProcess": "env",
    "labels": {
      "com.openfaas.scale.zero": "true",
      "com.openfaas.scale.zero-duration": "2m"
    },
    "annotations": {},
    "invocationCount": 46,
    "replicas": 0,
    "availableReplicas": 0,
    "createdAt": "2023-07-04T16:17:24Z"
  }
}
```

Note that there are 0 replicas (desired) and 0 available replicas (Pods) because the function is scaled to zero due to having been inactive for 2 minutes.

### Delete a function

```
FAAS_DEBUG=1 faas-cli remove env

Deleting: env.
DELETE https://127.0.0.1:8080/system/functions
Content-Type: [application/json]
User-Agent: [faas-cli/dev]
Authorization: [Bearer REDACTED]

{"functionName":"env"}
```

## Secrets

Secrets work in much the same way as functions, however you cannot retrieve the text or binary value of a secret after it has been created for security reasons.

To see the API calls for secrets, run `FAAS_DEBUG=1 faas-cli secret create` for instance.

```
POST http://127.0.0.1:8080/system/secrets
Content-Type: [application/json]
{"name":"api-key","namespace":"alex","value":"SECRET"}
```

List the functions within a namespace:

```
FAAS_DEBUG=1 faas-cli secret list -n alex
GET http://127.0.0.1:8080/system/secrets?namespace=alex
```

## Logs

Logs are provided in streaming JSON format and come from running Pods.

Therefore if a function is scaled to zero, there will be no logs available. To work around this, you may want to consider [an alternative logging provider](/architecture/logs-provider/) like Grafana Loki.

You can run `FAAS_DEBUG=1 faas-cli logs env` to see the API call to query the logs for a function.

```
FAAS_DEBUG=1 faas-cli logs env --tail --since 1h -o json

GET http://127.0.0.1:8080/system/logs
User-Agent: [faas-cli/dev]
Authorization: [Bearer REDACTED]
follow=true&name=env&since=2023-07-04T16%3A27%3A23%2B01%3A00&tail=-1
```

Example output when requesting JSON format:

```json
{"name":"env","namespace":"","instance":"env-5b6d68d94c-qqdtj","timestamp":"2023-07-04T16:32:22.223912519Z","text":"2023/07/04 16:32:22 Version: 0.1.4\tSHA: 86e85231a20df03bc9187a31c400f4bbc4e2b9ba\n"}
{"name":"env","namespace":"","instance":"env-5b6d68d94c-qqdtj","timestamp":"2023-07-04T16:32:22.223947899Z","text":"2023/07/04 16:32:22 Timeouts: read: 5s, write: 5s hard: 0s.\n"}
{"name":"env","namespace":"","instance":"env-5b6d68d94c-qqdtj","timestamp":"2023-07-04T16:32:22.223954179Z","text":"2023/07/04 16:32:22 Listening on port: 8080\n"}
{"name":"env","namespace":"","instance":"env-5b6d68d94c-qqdtj","timestamp":"2023-07-04T16:32:22.223983229Z","text":"2023/07/04 16:32:22 Writing lock-file to: /tmp/.lock\n"}
{"name":"env","namespace":"","instance":"env-5b6d68d94c-qqdtj","timestamp":"2023-07-04T16:32:22.223987809Z","text":"2023/07/04 16:32:22 Metrics listening on port: 8081\n"}
{"name":"env","namespace":"","instance":"env-5b6d68d94c-qqdtj","timestamp":"2023-07-04T16:32:23.380573646Z","text":"2023/07/04 16:32:23 Forking fprocess.\n"}
{"name":"env","namespace":"","instance":"env-5b6d68d94c-qqdtj","timestamp":"2023-07-04T16:32:23.382405318Z","text":"2023/07/04 16:32:23 Wrote 932 Bytes - Duration: 0.001823s\n"}
```

## Namespace management

OpenFaaS for Enterprises also includes REST endpoints for listing, creating, deleting and updating namespaces.

### Create a namespace with kubectl

To create a namespace and annotate it for OpenFaaS with `kubectl`, see: [Docs: Multiple Namespaces](/reference/namespaces/)

You can: list, create, update and delete namespaces with the REST API.

!!! Note "Note the difference in URL"
    The path for listing namespaces (read-only) is `/system/namespaces`, whilst mutating a single namespace will be either: `/system/namespace`, or `/system/namespaces/NAME`.

    The field for the body for mutations is `name`, rather than `namespace`.

### List namespaces

```bash
export OPENFAAS_URL="http://127.0.0.1:8080"
export TOKEN=""

curl -s \
  -H "Authorization: Bearer $TOKEN" \
    $OPENFAAS_URL/system/namespaces
```

### Create a namespace

Note that for initial creation, the namespace `n1` isn't included in the URL.

```bash
export OPENFAAS_URL="http://127.0.0.1:8080"
export TOKEN=""

curl -s \
  -X POST \
  --data-binary '{"name": "n1", "annotations": {"openfaas":"1"}}' \
  -H "Authorization: Bearer $TOKEN" \
    $OPENFAAS_URL/system/namespace/
```

Note that the trailing `/` slash in `/system/namespace/` is required for this endpoint, and that the name of the namespace must be passed within the body.

### Update a namespace

This endpoint can be used to update the annotations for an existing namespace, however the name cannot be changed.

```bash
export OPENFAAS_URL="http://127.0.0.1:8080"
export TOKEN=""

curl -s \
  -X PUT \
  --data-binary '{"name": "n1", "annotations": {"openfaas":"1", "customer": "openfaasltd"}}' \
  -H "Authorization: Bearer $TOKEN" \
    $OPENFAAS_URL/system/namespaces/n1
```

### Delete a namespace

```bash
export OPENFAAS_URL="http://127.0.0.1:8080"
export TOKEN=""

curl -s \
  -X DELETE
  -H "Authorization: Bearer $TOKEN" \
  --data-binary '{"name": "n1"}' \
    $OPENFAAS_URL/system/namespaces/n1
```

Only the name of the namespace needs to be included in the body for this endpoint.

## Anything else you'd like to know?

Please feel free to reach out if you're a customer using your existing channels or [join our weekly community call](/community/) to talk to us in person.
