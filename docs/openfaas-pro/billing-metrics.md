# Billing metrics

Collect detailed function usage metrics for billing customers or chargeback for internal teams via webhook events.

## Installation

To start receiving events with detailed usage metrics you need to enable metering and configure a webhook endpoint OpenFaaS can deliver events to. This can be done by editing the values.yaml file of the [OpenFaaS chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

```yaml
eventSubscription:
  endpoint: "https://example.com/openfaas-events"
  metering:
    enabled: true
    # The default memory value that is included in usage events if
    # no memory limit is set for a function.
    defaultRAM: 40Mi
```

## Validating webhook deliveries

If an endpoint secret is provided, OpenFaaS will use the secret token to create a hash signature of the webhook payload. The hash signature will be in the `X-Openfaas-Signature-256` header for each webhook delivery. Your code that handles the webhook deliveries should verify the signature to ensure the delivery in coming from OpenFaaS.

To configure the webhook secret you will need to generate a token and add it to your OpenFaaS cluster as a Kubernetes secret:

```bash
head -c 32 /dev/urandom | base64 | cut -d "-" -f1 > webhook-secret

kubectl create secret generic \
    -n openfaas \
    webhook-secret \
    --from-file webhook-secret=./webhook-secret
```

Reference the Kubernetes secret in the `eventSubscription` section in the `values.yaml` file.

```yaml
eventSubscription:
  endpointSecret: webhook-secret
```

## Receive metering events

OpenFaaS collects events from the OpenFaaS components in NATS JetStream. A separate components, the event-worker will process those events and post them to the configured webhook endpoint. The webhook can be used to process the events and store them in an SQL database or any other type of persistent storage.

!!! warning

    It is possible to use an OpenFaaS function to receive metering events. Take note that this function can not be deployed in the same OpenFaaS cluster that is sending the events as each invocation will in turn trigger a new event which will cause an infinite loop.

Events are send in batches of grouped events to reduce the number of webhook calls. The request payload is always a list of events.

### Delivery headers

- `X-Openfaas-Event`: The name of the event that triggered the delivery.
- `X-Openfaas-Signature-256`: This header is sent if the endpointSecret is configured in the Helm chart. This is the HMAC hex digest of the request body, and is generated using the SHA-256 hash function and the webhook secret as the HMAC key. It can be used to verify the origin of the webhook payload.
- `X-Openfaas-Delivery`: A unique ID identifying the webhook delivery.

### Example metering webhook delivery

```
> POST https://example.com/openfaas-events HTTP/2

> X-Openfaas-Event: function_usage
> X-Openfaas-Signature-256: sha256=d57c68ca6f92289e6987922ff26938930f6e66a2d161ef06abdf1859230aa23c
> X-Openfaas-Delivery: fe0f677c-c431-498e-8ace-9ba857434334
> Content-Type: application/json

> [
>   {
>     "event": "function_usage",
>     "namespace": "openfaas-fn",
>     "function_name": "env",
>     "stated": "2023-11-14T15:01:20.349527036Z",
>     "duration": 3798742,
>     "memory_bytes": 20971520
>   }
> ]
```

The request body contains a list of function usage events:

* `event` - the type of event
* `namespace` - the namespace of the function that triggered the event
* `function_name` - the function name that triggered the event
* `started` - the timestamp the function invocation was started
* `duration` - the duration of the function invocation
* `memory_bytes` - the memory limit configured for the function in bytes (See: [Memory/CPU limits](/reference/yaml/#function-memorycpu-limits)).
  If no memory limit is configured for a function a default value will be used. This default values can be configured in the
  Helm chart by setting the parameter `eventSubscription.metering.defaultRAM`.

## Use metering data

Once the event data is saved to a persistent data store it can be used to calculate billing metrics e.g the total number of invocations, total invocation duration and GB/s used by a function over a certain time period.

This example uses a PostgreSQL database to persist event data. In this case the data was inserted into a table `openfaas_metering.function_usage`.

It is possible to get detailed usage information per function over a certain time period. For example:

```
  namespace  | function_name | total_invocations | total_duration  |  total_gb_seconds  
-------------+---------------+-------------------+-----------------+--------------------
 openfaas-fn | sleep         |                43 | 00:01:26.742422 | 3.3883758593749995
 openfaas-fn | env           |             74724 | 00:29:09.566917 |  68.34245769531262
 openfaas-fn | figlet        |                 4 | 00:00:00.008232 |       0.0009953125
(3 rows)
```

An example of the PostgreSQL query that was used to get this data:

```sql
select
    u.namespace,
    u.function_name,
    count(u) as total_invocations,
    sum(u.duration) as total_duration,
    sum(extract(epoch from u.duration) * u.memory_bytes / power(1024,3)) as total_gb_seconds
from openfaas_metering.function_usage u
where u.started > now() - interval '30 days'
group by namespace,function_name;
```