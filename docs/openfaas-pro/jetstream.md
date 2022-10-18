# JetStream for OpenFaaS

The OpenFaaS async system used to be powered by NATS Streaming. The new generation of the [OpenFaaS async system](/reference/async) is backed by NATS JetStream.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

## Async use cases
Async can be used for any OpenFaaS function invocation, where the response is not required immediately, but is either discarded or made available at a later time. Some use-cases include:

- Batch processing and machine learning
- Resilient data pipelines
- Receiving webhooks
- Long running jobs

On our blog we demo and explore some architectural patterns for these uses cases:

- [Exploring the Fan out and Fan in pattern with OpenFaaS](https://www.openfaas.com/blog/fan-out-and-back-in-using-functions/)
- [Generate PDFs at scale on Kubernetes using OpenFaaS and Puppeteer](https://www.openfaas.com/blog/pdf-generation-at-scale-on-kubernetes/)
- [How to process your data the resilient way with back pressure](https://www.openfaas.com/blog/limits-and-backpressure/)

## Features

### Metrics an monitoring

Get insight in the behaviour of your queues with built in metrics.

Prometheus metrics are available for monitoring things like the number of messages that have been submitted to the queue over a period of time, how many messages are waiting to be completed and the total number of messages that where processed.

An overview of all the available metrics can be found in the [metrics reference](/architecture/metrics/#jetstream-for-openfaas)

![Grafana dashboard for the queue-worker](https://www.openfaas.com/images/2022-07-jetstream-for-openfaas/queue-worker-dashboard.png)
> Grafana dashboard for the queue-worker

### Multiple queues

OpenFaaS ships with a “mixed queue”, where all invocations run in the same queue. If you have special requirements, you can set up your own separate queue and queue-worker using the [queue-worker helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/queue-worker).

See: [multiple queues](/reference/async/#multiple-queues)

### Retries
Users can specify a list of HTTP codes that should be retried a number of times using an exponential back-off algorithm to mitigate the impact associated with retrying messages. 

See: [retries](/openfaas-pro/retries)

### Structured JSON logging

Logs from the queue-worker can be formatted for readability, during development, or in JSON for a log aggregator like ELK or Grafana Loki.

You can change the logging format by editing the values.yaml file for the [OpenFaaS chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas)

```yaml
jetstreamQueueWorker:
  logs:
    format: json
```

![Structured logs formatted for the console](https://www.openfaas.com/images/2022-07-jetstream-for-openfaas/structured-logs.png)
> Structured logs formatted for the console

## See also

- [The Next Generation of Queuing: JetStream for OpenFaaS](https://www.openfaas.com/blog/jetstream-for-openfaas/)
