# Queue Worker

The Queue Worker is a batteries-included, scale-out queue for invoking functions asynchronously.

This page is primarily concerned with how to configure the Queue Worker, you can learn about [asynchronous invocations here](/reference/async).

> Note: This feature is included for [OpenFaaS Standard & For Enterprises](https://openfaas.com/pricing/) customers.

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

* NATS - an open source messaging system hosted by the [CNCF](https://www.cncf.io/)
* NATS JetStream - a messaging system built on top of NATS for durable queues and message streams

  1. A JetStream Server is the NATS server, running in *jetstream* mode
  2. A Stream is a message store it is used in OpenFaaS to queue async invocation messages.
  3. A Consumer is a stateful view of a stream when clients consume messages from a stream the consumer keeps track of which messages were delivered and acknowledged.
  5. A Subscriber is what the queue-worker creates to start pulling messages from the stream.

Learn more about [NATS JetStream](https://docs.nats.io/nats-concepts/jetstream)

## Installation

For staging and development environments OpenFaaS can be deployed with an embedded version of the NATS server which uses an in-memory store.

To enable JetSteam for OpenFaaS set `jetstream` as the queue mode in the values.yaml file of the [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md)

```yaml
queueMode: jetstream
nats:
    streamReplication: 1
```

If the NATS Pod restarts, you will lose all messages that it contains. In your development or staging environment. This could happen if you update the chart and the version of the NATS server has changed, or if a node is removed from the cluster.

For production environments you should install NATS separately using its Helm chart.

NATS can be configured with a quorum of at least 3 replicas so it can recover data if one of the replicas should crash. You can also enable a persistent volume in the NATS chart for additional durability. 

If you are running with 3 replicas of the NATS server, then update the OpenFaaS chart to reflect that in the `nats.streamReplication` parameter. With this in place, the stream for queued messages will be replicated across the 3 NATS servers.

```yaml
queueMode: jetstream
nats:
  streamReplication: 3
  external:
    enabled: true
    host: "nats.nats"
    port: "4222"
```

By default the NATS helm chart will be installed into the nats namespace with the name of `nats`, but you can customise this if you wish by setting the `nats.external.host` parameter.

## Features

### Queue-based scaling for functions

The queue-worker uses a shared NATS Stream and NATS Consumer by default, which works well with many of the existing [autoscaling strategies](/reference/async/#autoscaling).

However, if you wish to scales functions based upon the queue depth for each, you can set up the queue-worker to scale its NATS Consumers dynamically for each function.

```yaml
jetstreamQueueWorker:
  mode: static | function
  consumer:
    inactiveThreshold: 30s
```

The `mode` parameter can be set to `static` or `function`.

If set to `static`, the queue-worker will scale its NATS Consumers based upon the number of replicas of the queue-worker. This is the default mode, and ideal for development, or constrained environments.

If set to `function`, the queue-worker will scale its NATS Consumers based upon the number of functions that are active in the queue. This is ideal for production environments where you want to scale your functions based upon the queue depth. It also gives messages queued at different times a fairer chance of being processed earlier.

The `inactiveThreshold` parameter can be used to set the threshold for when a function is considered inactive. If a function is inactive for longer than the threshold, the queue-worker will delete the NATS Consumer for that function.

### Metrics and monitoring

Get insight into the behaviour of your queues with built in metrics.

Prometheus metrics are available for monitoring things like the number of messages that have been submitted to the queue over a period of time, how many messages are waiting to be completed and the total number of messages that were processed.

An overview of all the available metrics can be found in the [metrics reference](/architecture/metrics/#jetstream-for-openfaas)

![Grafana dashboard for the queue-worker](https://www.openfaas.com/images/2022-07-jetstream-for-openfaas/queue-worker-dashboard.png)
> Grafana dashboard for the queue-worker

### Multiple queues

OpenFaaS ships with a “mixed queue”, where all invocations run in the same queue. If you have special requirements, you can set up your own separate queue and queue-worker using the [queue-worker helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/queue-worker).

See: [multiple queues](/reference/async/#multiple-queues)

> If the capacity of your queue does not fit within the default limits described [here](#configure-jetstream) you will need to follow these steps to [create a Stream and Consumer manually](#configure-streams-and-consumers-manually) for each queue.

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

## Configure JetStream

Every OpenFaaS async queue requires a Stream and Consumer to be created on the JetStream server. By default the queue-worker manages these for you and ensures they are created on startup if they do not exist.

Stream and Consumers can be managed manually although it is not recommended.

### Configure Streams and Consumers manually

Streams and Consumers can be defined manually, typically using the [NATS CLI tool](https://docs.nats.io/running-a-nats-service/configuration/resource_management/configuration_mgmt/nats-admin-cli). 

>A [Kubernetes controller](https://docs.nats.io/running-a-nats-service/configuration/resource_management/configuration_mgmt/kubernetes_controller) is available for managing Streams and Consumers declaratively. This can be used if you are using a CD system like ArgoCD or Flux.

You can use [arkade](https://github.com/alexellis/arkade) to install the NATS CLI.
```
arkade get nats
```

Port forward the nats server to your localhost so you can use the cli to interact with it.
```bash
kubectl port-forward -n openfaas svc/nats 4222:4222
```
#### Create a Stream

The Stream will need to be created first. In this example we will create the Stream for the shared queue, `faas-request`. We recommend giving the stream the same name as the queue.

```bash
export QUEUE_NAME=faas-request

nats stream create $QUEUE_NAME \
  --subjects=$QUEUE_NAME \
  --replicas=1 \
  --retention=work \
  --discard=old \
  --max-msgs-per-subject=-1
```

Messages intended for a queue are published to a NATS subject, `faas-request` by default and the queue name for a [dedicated queue](/reference/async/#multiple-queues). A Stream should only bind the subject for the queue that it will be associated with. This can be configured with the `--subjects` flag.

The `--replicas` flag is used to configure the stream replication factor. This should be at least 3 for production environments.

The command above includes the required settings for using the Stream with a queue-worker. The queue-worker requires a retention policy of type `work (WorkQueuePolicy)` . You will get prompted interactively for the remaining stream information. Use the defaults or configure your own storage and limits for the stream.

```
? Storage file
? Stream Messages Limit -1
? Total Stream Size -1
? Message TTL -1
? Max Message Size -1
? Duplicate tracking time window 2m0s
? Allow message Roll-ups No
? Allow message deletion Yes
? Allow purging subjects or the entire stream Yes
``` 

#### Create a Consumer

Once the Stream has been created the Consumer can be added. We recommend naming giving the consumer the same name as the queue with the suffix `-workers` added to it. The consumer for the shared `faas-request` queue would be named `faas-request-workers`.

```bash
export QUEUE_NAME=faas-request

nats consumer \
  create $QUEUE_NAME $QUEUE_NAME-workers \
  --pull \
  --deliver=all \
  --ack=explicit \
  --replay=instant \
  --max-deliver=-1 \
  --max-pending=-1 \
  --no-headers-only \
  --backoff=none \
  --wait=3m \
  --max-waiting=512 \
  --defaults
```

This command creates a pull consumer that makes available all messages for every subject on the stream. We require that each message is acknowledged explicitly.

Important configuration flags:
  - The queue-worker will control how many times a message can be redelivered,`--max-deliver` has to be set to `-1` to allow unlimited deliveries.

  - The queue-worker automatically extends the ack window for functions that require more time to complete. In order to prevent us from having to extend the ack window to often we recommend configuring a default acknowledgement waiting time of 3 minutes. This can be configured with the `--wait` flag.

  - The `--max-pending` flag limits the number of messages that can have a pending status. Messages that are queued for retries are also considered pending. It is set to `-1` to allow an unlimited number of pending messages by default. The consumer is paused and no new messages are delivered when this limit is reached. If you select a customer value it should be at least `max_inflight * queue-worker-replicas + buffer`. The size of the buffer depends on the number of retries your queue needs to be able to handle.

    This value can always be update later:

    ```bash
    nats consumer edit --max-pending 6000
    ```
  
  - The `--max-waiting` flag limits the number of subscribers a queue-worker can create. We recommend using the default value of 512 but the value should be at least equal to the number of queue worker replicas.

#### Configure the queue-worker
As a final step the queue-worker needs to be configured to use the externally created Stream and Consumer.

For the shared OpenFaaS queue edit the values.yaml file of the OpenFaaS chart. The name of the Consumer used by the queue-worker is set with `jetstreamQueueWorker.durableName`. The name of the Stream needs to be set with `nats.channel`.

```yaml
jetstreamQueueWorker:
  durableName: faas-workers

nats:
  streamReplication: 1
  channel: faas-request-workers
```

A dedicated queue using the [queue-worker Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/queue-worker) can be configured by setting the `nats.stream.name` and `nats.consumer.durableName` parameters.

## Reset the stream for async messages

From time to time, you may wish to reset or purge the stream for async messages. Either due to a configuration change in the stream that can not be applied automatically, or because you have generated a large number of unnecessary messages you want to remove from the queue.

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
    # This example deletes the default queue-worker stream.
    # Replace the stream name for dedicated queues
    # deployed with the queue-worker Helm chart.
    export STREAM_NAME=faas-request

    nats stream delete $STREAM_NAME
    ```

4. Restart the queue-worker deployment.

    ```bash
    kubectl rollout restart -n openfaas deploy/queue-worker
    ```

## See also

- [The Next Generation of Queuing: JetStream for OpenFaaS](https://www.openfaas.com/blog/jetstream-for-openfaas/)
