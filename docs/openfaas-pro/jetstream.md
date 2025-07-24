# Queue Worker

> Note: This feature is included for [OpenFaaS Standard & For Enterprises](https://openfaas.com/pricing/) customers.

The Queue Worker is part of the built-in queue system for OpenFaaS built upon NATS JetStream.

It's a batteries-included solution, that's used at scale by many OpenFaaS customers in production every day.

Async invocations can be submitted over HTTP by your own code or through an event-connector.

This page is primarily concerned with how to configure the Queue Worker.

You can learn about [asynchronous invocations here](/reference/async).

## Async use cases

Every function in OpenFaaS can be invoked either synchronously or asynchronously. Asynchronous invocations are retried automatically, and can return their response to a given endpoint via a webhook.

Popular use-cases include:

- Batch processing and machine learning
- Resilient data pipelines
- Receiving webhooks
- Long running jobs

On the blog we show reference examples built upon these architectural patterns:

- [Exploring the Fan out and Fan in pattern with OpenFaaS](https://www.openfaas.com/blog/fan-out-and-back-in-using-functions/)
- [Generate PDFs at scale on Kubernetes using OpenFaaS and Puppeteer](https://www.openfaas.com/blog/pdf-generation-at-scale-on-kubernetes/)
- [How to process your data the resilient way with back pressure](https://www.openfaas.com/blog/limits-and-backpressure/)
- [The Next Generation of Queuing: JetStream for OpenFaaS](https://www.openfaas.com/blog/jetstream-for-openfaas/)

## Terminology

* [NATS](https://nats.io/) - an open source messaging system hosted by the [CNCF](https://www.cncf.io/)
* [NATS JetStream](https://docs.nats.io/nats-concepts/jetstream) - a messaging system built on top of NATS for durable queues and message streams

1. A JetStream Server is the original NATS Core project, running in "jetstream mode"
2. A Stream is a message store it is used in OpenFaaS to queue async invocation messages.
3. A Consumer is a stateful view of a stream when clients consume messages from a stream the consumer keeps track of which messages were delivered and acknowledged.

## Installation

**Embedded NATS server**

For staging and development environments OpenFaaS can be deployed with an embedded version of the NATS server which uses an in-memory store. This is the default when you install OpenFaaS using the [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md).

If the NATS Pod restarts, you will lose all messages that it contains. This could happen if you update the chart and the version of the NATS server has changed, or if a node is removed from the cluster.

** External NATS server**
For production environments you should install NATS separately using its Helm chart.

NATS can be configured with a quorum of at least 3 replicas so it can recover data if one of the replicas should crash. You can also enable a persistent volume in the NATS chart for additional durability.

If you are running with 3 replicas of the NATS server, then update the OpenFaaS chart to reflect that in the `nats.streamReplication` parameter. With this in place, the stream for queued messages will be replicated across the 3 NATS servers.

```yaml
nats:
  streamReplication: 3
  external:
    enabled: true
    host: "nats.nats"
    port: "4222"
```

By default the NATS helm chart will be installed into the nats namespace with the name of `nats`, but you can customise this if you wish by setting the `nats.external.host` parameter.

Instructions for a recommended NATS production deployment are available for customers though the [customer community repo](https://github.com/openfaas/customers/blob/master/jetstream.md)

## Features

### Queue-based scaling for functions

The queue-worker uses a shared NATS Stream and NATS Consumer by default, which works well with many of the existing [autoscaling strategies](/reference/async/#autoscaling). Requests are processed in a FIFO order, and it is possible for certain functions to dominate or starve the queue.

A fairer approach is to scale functions based upon their respective queue depth, with a consumer created for each function as and when it is needed.

The `mode` parameter can be set to `static` (default) or `function`.

```yaml
jetstreamQueueWorker:
  mode: static | function
  consumer:
    inactiveThreshold: 30s
```

* If set to `static`, the queue-worker will scale its NATS Consumers based upon the number of replicas of the queue-worker. This is the default mode, and ideal for development, or constrained environments.

* If set to `function`, the queue-worker will scale its NATS Consumers based upon the number of functions that are active in the queue. This is ideal for production environments where you want to [scale your functions based upon the queue depth](/reference/autoscaling/). It also gives messages queued at different times a fairer chance of being processed earlier.

* The `inactiveThreshold` parameter can be used to set the threshold for when a function is considered inactive. If a function is inactive for longer than the threshold, the queue-worker will delete the NATS Consumer for that function.

### Metrics and monitoring

Get insight into the behaviour of your queues with built in metrics.

Prometheus metrics are available for monitoring things like the number of messages that have been submitted to the queue over a period of time, how many messages are waiting to be completed and the total number of messages that were processed.

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

## Clear or reset a queue

From time to time, you may wish to reset or purge an async message queue. You may have generated a large number of unnecessary messages that you want to remove from the queue.

[Cancel async invocations](/reference/async/#cancel-async-invocations)

Each queue-worker deployment creates a dedicated NATS Stream to store async invocation messages. On startup the queue-worker will create this stream if it does not exist. To reset Keep in mind that all queued invocations will be lost.

The stream is automatically created by the queue-worker of it does not exist. Simply deleting the stream and restarting the queue-worker will reset the stream and consumer.

1. Get the NATS CLI

    [Download the NATS CLI](https://github.com/nats-io/natscli/releases/) or use [arkade](https://github.com/alexellis/arkade) to install it.

    ```
    arkade get nats
    ```

2. Port forward the NATS service.

    ```bash
    kubectl port-forward -n openfaas svc/nats 4222:4222
    ```

3. Delete the queue-worker stream.

    Keep in mind that deleting the stream removes any queued async invocations.

    ```bash
    # This example deletes strean for the OpenFaaS default queue named faas-request.
    # The stream name is always the queue name prefixed with OF_.
    # Replace the queue name with the name of the queue stream you want to delete.
    export QUEUE_NAME=faas-request

    nats stream delete OF_$QUEUE_NAME
    ```

4. Restart the queue-worker deployment.

    ```bash
    kubectl rollout restart -n openfaas deploy/queue-worker
    ```

## See also

- [The Next Generation of Queuing: JetStream for OpenFaaS](https://www.openfaas.com/blog/jetstream-for-openfaas/)
