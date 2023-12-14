# Billing metrics

With the *metering* feature, detailed metrics are made available for function invocations. These can be used to implement your own billing system.

Use-cases:

* Billing SaaS customers - when you are hosting functions, bill customers based upon usage
* Chargeback for internal teams - charge internal teams for a proportion of their use of an internal functions service

After each function has completed an invocation, a summary is published to NATS JetStream, this is then collected by a resilient and scalable event-worker. The event-worker will collect the events and send them in groups to a webhook endpoint for efficiency.

Events are batched within each HTTP call to your webhook endpoint, and will also contain a HMAC header that can be used to verify the origin of the event.

!!! info "OpenFaaS Enterprise feature"
    This feature is included for [OpenFaaS Enterprise](/openfaas-pro/introduction) customers.

## Installation

To start receiving events with detailed usage metrics you need to enable metering and configure a webhook endpoint OpenFaaS can deliver events to. This can be done by editing the values.yaml file of the [OpenFaaS chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

You will need to create an `endpointSecret` which will be shared with the HTTP receiver, and used to sign the webhook payload.

```bash
# If openssl is not available on your system, use the following:
head -c 32 /dev/urandom | base64 | cut -d "-" -f1 > billing-endpoint-secret.txt

# openssl is preferred to generate a random secret:
openssl rand -base64 32 > billing-endpoint-secret.txt

kubectl create secret generic \
    -n openfaas \
    webhook-secret \
    --from-file webhook-secret=./billing-endpoint-secret.txt
```

Next, update your copy of the `values.yaml` file for the main OpenFaaS chart:

```yaml
eventSubscription:
  endpoint: "https://example.com/openfaas-events"
  endpointSecret: webhook-secret

  metering:
    enabled: true
    defaultRAM: 512Mi
```

* The `eventSubscription` section is also used to configure auditing.
* Set the `endpoint` to your HTTP endpoint that will receive the events, including any Path you want to include.
* The `endpointSecret` is used to sign the webhook payload with a symmetric secret using HMAC and a 256-bit digest.
* The defaultRAM under the metering secret is expressed in the same notation as Kubernetes Pods, i.e. `128Mi` or `1Gi`. This value is used when there is no memory limit configured on a function.

When you're ready, apply the changes to your cluster using `helm upgrade --install` and your values.YAML file.

## The webhook receiver endpoint

You can modify an existing part of your platform to expose a new HTTP path, or deploy a new HTTP microservice to collect the webhooks.

### How to use a function to receive webhooks

We recommend recording billing information by a service outside of the OpenFaaS cluster.

However, with additional configuration, a function can be used so long as it is in a separate namespace, and is configured in the Helm chart to be excluded from metering.

```yaml
metering:
  excludeNamespaces: "openfaas-system"
```

Without this configuration, an infinite loop would occur, as the function receiving the metering events, would itself also be metered, and therefore trigger itself.

### How to validate the webhook is genuine

The event-worker will use the `endpointSecret` to create a hash signature of the webhook's payload. The hash signature will be in the `X-Openfaas-Signature-256` header for each webhook delivery. Your handler will need to verify the signature matches the payload before processing any events.

### Webhook delivery headers

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
>     "started": "2023-11-14T15:01:20.349527036Z",
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

## Example: Persisting event data to PostgreSQL

Once the event data is saved to a persistent data store it can be used to calculate billing metrics e.g the total number of invocations, total invocation duration and Gigabyte seconds (GB/s) used by a function over a certain time period.

This example uses a PostgreSQL database to persist event data. In this case the data was inserted into a table `openfaas_metering.function_usage`.

Assuming the following table schema:

```sql
CREATE TABLE IF NOT EXISTS
	openfaas_metering.function_usage (
		id SERIAL PRIMARY KEY,
		namespace TEXT NOT NULL,
		function_name TEXT NOT NULL,
		duration interval NOT NULL,
		started_at TIMESTAMP NOT NULL,
		ram_bytes BIGINT NOT NULL,
		created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
	);
```

Example insert statement:

```golang
	insert, err := tx.Prepare(`INSERT INTO
	openfaas_metering.function_usage (namespace, function_name, duration, started_at, ram_bytes)
	VALUES
	($1, $2, make_interval(secs => $3::NUMERIC / 1000000000), $4, $5)`)
```

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

## Q&A

* Where is the metering data collected from?

    The OpenFaaS gateway is responsible for collecting the metering data, then when a buffer fills up or a timer goes off, the data will be flushed from memory to NATS JetStream.

* Will metering add overhead to my function invocations?

    Events are published asynchronously to NATS JetStream, so should not add any overhead to the function invocation itself.

* How many events will be sent within a webhook?

    The amount of events are 1 to many, depending on how many were available when they were published to NATS JetStream from the OpenFaaS gateway.

* How much load will the event-worker generate on the cluster?

    This depends on the rate of invocations on your cluster, it's unlikely to be noticeable with moderate traffic.

* Can the events be written to our database?

    You can write a webhook receiver and use it to perform inserts into the database. If your chosen datastore supports batch inserts, we would recommend using that instead of inserting each event individually. We have provided an example schema and query for calculating billing metrics from PostgreSQL.

## Do you have questions, comments, or suggestions?

Feel free to reach out to the team [for a call](https://openfaas.com/pricing).
