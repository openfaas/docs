# Auditing

Record activities and changes made through the OpenFaaS REST API.

Use-cases:

* Maintain access records for compliance and security.
* Monitor activity on the OpenFaaS REST API.

When auditing is enabled, OpenFaaS dispatches events containing detailed information about each request  made to the OpenFaaS REST API to a designated webhook endpoint. After each request to the API 

Events are batched within each HTTP call to your webhook endpoint, and will also contain a HMAC header that can be used to verify the origin of the event.

!!! info "OpenFaaS Enterprise feature"
    This feature is included for [OpenFaaS Enterprise](/openfaas-pro/introduction) customers.
    [Identity and Access Management](/openfaas-pro/iam/overview/) needs to be enabled for this feature.


## Installation

To start receiving auditing events you need to enable auditing and configure a webhook endpoint OpenFaaS can deliver events to. This can be done by editing the values.yaml file of the [OpenFaaS chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

You will need to create an `endpointSecret` which will be shared with the HTTP receiver, and used to sign the webhook payload.

```bash
head -c 32 /dev/urandom | base64 | cut -d "-" -f1 > event-webhook-secret.txt

kubectl create secret generic \
    -n openfaas \
    webhook-secret \
    --from-file webhook-secret=./event-webhook-secret.txt
```

Next, update your copy of the `values.yaml` file for the main OpenFaaS chart:

```yaml
eventSubscription:
  endpoint: "https://example.com/openfaas-events"
  endpointSecret: webhook-secret

  auditing:
    enabled: true
```

* The `eventSubscription` section is also used to configure metering.
* Set the `endpoint` to your HTTP endpoint that will receive the events, including any Path you want to include.
* The `endpointSecret` is used to sign the webhook payload with a symmetric secret using HMAC and a 256-bit digest.


## Example auditing webhook delivery

Each webhook delivery will contain a set of headers to identify events and validate the payload. See: [Webhook delivery headers](/openfaas-pro/billing-metrics/#webhook-delivery-headers).

A webhook delivery can be validated using a shared HMAC secret and the payload signature included in the 
`X-Openfaas-Signature-256` header for each request. See: [How to validate the webhook is genuine.](/openfaas-pro/billing-metrics/#installation)

```
> POST https://example.com/openfaas-events HTTP/

> X-Openfaas-Event: api_access
> X-Openfaas-Signature-256: sha256=d57c68ca6f92289e6987922ff26938930f6e66a2d161ef06abdf1859230aa23c
> X-Openfaas-Delivery: fe0f677c-c431-498e-8ace-9ba857434334
> Content-Type: application/json

> 
> [
>   {
>     "actor": {
>       "sub": "fed:system:serviceaccount:openfaas:cron-connector",
>       "issuer": "https://gw.exit.welteki.dev",
>       "fed_issuer": "https://kubernetes.default.svc.cluster.local"
>     },
>     "path": "/system/namespaces",
>     "method": "GET",
>     "actions": [
>         "Namespace:List"
>     ],
>     "response_code": 200,
>     "time": "2023-12-04T17:09:28.710118072Z"
>   }
> ]
```

The request body contains a list of api access events.

* `actor` - the user that triggered the event. This field will be empty if the request is unauthenticated.
    * `sub` - OIDC subject, a unique identifier for the user
    * `name` - the full name of the subject (when available in the OIDC claims)
    * `issuer` - the OpenFaaS issuer that issued the access token
    * `fed_issuer` - the federated issuer that issued to initial access token
* `path` - the API path
* `method` - the request method
* `actions` - the applied action e.g. `Namespace:List`, `Function:List`, etc. (See: [IAM Permissions](https://docs.openfaas.com/openfaas-pro/iam/overview/#permissions))
* `response_code` - the response code returned by the request
* `time` - the time the API request started