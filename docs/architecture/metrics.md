# Monitoring Functions

## Introduction

All OpenFaaS metrics are exposed in Prometheus format, and collected by a built-in Prometheus server which is deployed via the OpenFaaS Helm chart.

There are two main uses for the built-in Prometheus server:

1. To power scale to zero, and the horizontal Pod autoscaler.
2. To provide basic metrics to end-users, and to power the Grafana dashboards offered to OpenFaaS Standard customers.

### Viewing metrics

See the various [Grafana dashboards](/openfaas-pro/grafana-dashboards) curated by our team.

### Long term retention of metrics

* There is no persistence in the Prometheus Pod, so restarting the Prometheus will remove all historic metrics. This is as designed, since the metrics are collected for autoscaling and short-term monitoring.
* The default retention period is 15 days, so anything older than that will no longer be visible. This is as designed, however the Helm chart does offer a way to modify this if disk space is becoming an issue, or you need to retain metrics for slightly longer.

What if you would like to enable long-term retention of Prometheus metrics?

Our recommendation is *not to* try to re-configure or alter the built-in Prometheus server, but to deploy your own, and to scrape the internal one via [Prometheus Federation](https://prometheus.io/docs/prometheus/latest/federation/).

With Prometheus Federation, you can also specify which series to collect, rather than collecting everything, which is more efficient for storage and works out cheaper in the long run.

Long-term retention can be achieved via the upstream Prometheus project using its Helm chart/operator, a SaaS version hosted by a cloud provider, [Grafana Cloud's agent](https://grafana.com/grafana/), or a solution built on top of Prometheus like [Thanos](https://github.com/thanos-io/thanos) or [Cortex](https://github.com/cortexproject/cortex).

## Gateway

The Gateway component exposes several metrics to help you monitor the health and behavior of your functions.

The Community Edition exposes basic metrics, with OpenFaaS Pro extending on the data available.

| Metric                              | Type       | Description                         | Labels                     | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | -------------------------- |--------------------|
| `gateway_functions_seconds`         | histogram  | Function invocation time taken      | `function_name`            | Community Edition  |
| `gateway_function_invocation_total` | counter    | Function invocation count           | `function_name`, `code`    | Community Edition  |
| `gateway_service_count`             | gauge    | Number of function replicas         | `function_name`            | Community Edition  |

Advanced metrics for OpenFaaS Pro users:

| Metric                              | Type       | Description                         | Labels                     | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | -------------------------- |--------------------|
| `gateway_invocation_function_started`             | counter    | Invocations started, including async | `function_name`            | Pro Edition  |
| `gateway_invocation_function_invocation_inflight`             | gauge    | Total connections inflight for function invocations | `function_name`            | Pro Edition  |
| `gateway_service_ready_count`             | gauge    | Number of function replicas which are in a ready state | `function_name`            | Pro Edition  |
| `gateway_service_target`            | gauge      | Target load for the function        | `function_name`            | Pro Edition  |
| `gateway_service_min`               | gauge      |  Min number of function replicas    | `function_name`            | Pro Edition  |
| `http_request_duration_seconds`     | histogram  | Seconds spent serving HTTP requests | `method`, `path`, `status` | Pro Edition  |
| `http_requests_total`               | counter    | The total number of HTTP requests   | `method`, `path`, `status` | Pro Edition  |
| `http_requests_total`               | counter    | The total number of HTTP requests   | `method`, `path`, `status` | Pro Edition  |

The `http_request*` metrics record the latency and statistics of `/system/*` routes to monitor the OpenFaaS gateway and its provider. The `/async-function` route is also recorded in these metrics to observe asynchronous ingestion rate and latency.

Additional metrics from the Operator:

| Metric                              | Type       | Description                         | Labels                     | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | -------------------------- |--------------------|
| `faasnetes_scale_total`                  | counter    | Number of times a function has been scaled (ignoring requests where current and desired replicas are equal) | `function_name`, `status` | Pro Edition  |
| `faasnetes_sync_handler_gauge`             | gauge      | Number of reconciliation functions running at given time | `status` | Pro Edition  |
| `faasnetes_sync_handler_histogram`        | histogram  | Time taken to reconcile function Custom Resources into Kubernetes objects | `status` | Pro Edition  |

The `faasnetes_scale_total` metric is useful for tracking the number of times a function has been scaled up or down. The `faasnetes_sync_handler_gauge` and `faasnetes_sync_handler_histogram` metrics are useful for tracking the amount of time spent reconciling function Custom Resources into Kubernetes objects in large deployments of OpenFaaS.

## CPU & RAM usage/consumption

CPU & RAM usage/consumption metrics are available for OpenFaaS Pro users via Prometheus and the OpenFaaS REST API, OpenFaaS Pro Dashboard and OpenFaaS CLI via `faas-cli describe`.

| Metric                              | Type       | Description                         | Labels                     | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | -------------------------- |--------------------|
| `pod_cpu_usage_seconds_total`         | counter  | CPU seconds consumed by all the replicas of a given function | `function_name`, `namespace`| Pro Edition  |
| `pod_memory_working_set_bytes`         | gauge  | Bytes of RAM consumed by all the replicas of a given function | `function_name`, `namespace`| Pro Edition  |

## JetStream for OpenFaaS

The queue-worker for NATS JetStream exposes metrics to help you get insight in the behavior of your OpenFaaS queues.

| Metric                                  | Type       | Description                                     | Labels       | Edition     |
| ----------------------------------------| ---------- | ------------------------------------------------| -------------|-------------|
| `queue_worker_pending_messages`         | gauge      | Amount of messages waiting to be processed. | `queue_name`, `function_name` | Pro Edition |
| `queue_worker_messages_processed_total` | counter    | Total number of messages processed              | `queue_name`, `kubernetes_pod_name` | Pro Edition |
| `queue_worker_messages_submitted_total` | gauge      | Total number of messages submitted to the queue by the gateway | `queue_name`, `kubernetes_pod_name` | Pro Edition |
| `queue_worker_function_invocation_inflight` | gauge |  Total number of inflight function requests made by the queue-worker | `queue_name`, `function_name` | Pro Edition |

## Watchdog

The classic and of-watchdog both provide Prometheus instrumentation on TCP port 8081 on the path /metrics. This is to enable the use-case of HPAv2 from the Kubernetes ecosystem.

| Metric                              | Type       | Description                         | Labels                       | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | ---------------------------- |--------------------|
| `http_request_duration_seconds`     | histogram  | Seconds spent serving HTTP requests | `method`, `path`, `status`   | Community Edition  |
| `http_requests_total`               | counter    | The total number of HTTP requests   | `method`, `path`, `status`   | Community Edition  |
| `http_requests_in_flight`           | gauge      | The number of HTTP requests in flight | `method`, `path`, `status` | Pro Edition        |

## Provider

The [FaaS Provider](/architecture/faas-provider) is the back-end API used by other OpenFaaS components like the Gateway. It exposes several metrics.

| Metric                              | Type       | Description                         | Labels                     | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | ---------------------------|--------------------|
| `provider_http_request_duration_seconds`     | histogram  | Seconds spent serving HTTP requests | `method`, `path`, `code`   | Pro Edition  |
| `provider_http_requests_total`               | counter    | The total number of HTTP requests   | `method`, `path`, `code`   | Pro Edition  |

The `http_request*` metrics record the latency and statistics of `/system/*` routes. Part of this information is also recorded in the metrics for the Gateway component. The purpose of exposing separate metrics on the provider component is to show the count of calls, to show efficiency, and to show the duration for performance testing, along with errors to flag unseen issues.

## Kafka connector

| Metric                              | Type       | Description                         | Labels                     | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | ---------------------------|--------------------|
| `kafka_connector_consumer_group_connect"`     | counter  | Total times the consumer group has attempted to connect to the broker | `group` | Pro Edition  |
| `kafka_connector_messages_consumed`               | counter    | Total messages received from the broker   | `group`, `topic`, `partition`, `member_id`  | Pro Edition  |
| `kafka_connector_messages_offset`               | gauge    | Offset committed  | `group`, `topic`, `partition`, `member_id`    | Pro Edition  |

## RabbitMQ connector

| Metric                              | Type       | Description                         | Labels                     | Edition            |
| ----------------------------------- | ---------- | ----------------------------------- | ---------------------------|--------------------|
| `rabbitmq_connector_messages_processed_total`     | counter | Total number of messages processed | `queue` | Pro Edition  |
| `rabbitmq_connector_queue_depth`               | gauge    | The total number of HTTP requests   | `queue` | Pro Edition  |

## Example queries for dashboarding

OpenFaaS Pro customers have access to 4 different dashboards which we've co-designed with our users, you can find out more in the [comparison page of OpenFaaS CE vs Pro](/openfaas-pro/introduction)

These basic metrics can be used to track the health of your functions as well a general usage patterns. See the Prometheus [documentation][prom-query-basics] and [examples][prom-query-examples] for more details about the available options and query functions. Below are several queries you might want to include in a basic [Grafana](https://grafana.com) dashboard for observing your OpenFaaS functions

### Function invocation rate

Return the per-second rate of invocation as measured over the previous 1 minute:

```
rate ( gateway_function_invocation_total [1m])
```

### Function replica count / scaling

Return the total function replicas:

```
gateway_service_count
```

### Total OK Function Invocation

Return the total number of successful function invocations:

```
sum( gateway_function_invocation_total {  code=\"200\"}
```

### Function execution time

Return the average function execution time, as measure over the previous 20 seconds:

```
(rate(gateway_functions_seconds_sum[20s]) / rate(gateway_functions_seconds_count[20s]))
```

### Metrics for a single function

Each of the metrics generated by the Gateway are labeled with and can be filtered by the function name,  For example The invocation rate for just a single function (e.g. if the function name is `echo`) is given by

```
rate ( gateway_function_invocation_total{function_name='echo'} [20s])
```

[prom-query-basics]: https://prometheus.io/docs/prometheus/latest/querying/basics/
[prom-query-examples]: https://prometheus.io/docs/prometheus/latest/querying/examples/
