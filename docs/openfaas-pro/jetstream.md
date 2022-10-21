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

## Terminology

In JetStream ("js" for short), there are new terms that will help us all in running and debugging the product.

1. A JetStream Server is the original NATS Core project, running in "jetstream mode"
2. A Stream is a set of messages, and works in a similar way to Kafka
3. A Consumer is a group that various Subscribers can join to read messages and keep track of where they left off
5. A Subscriber is what the queue-worker creates, if the max_inflight is set to 25, the queue-worker will create 25 subscribers

> You can learn more about JetStream here: [Docs for JetStream](https://docs.nats.io/nats-concepts/jetstream)

## Installation

For staging and development environments OpenFaaS can be deployed with an embedded version of the NATS server which uses an in-memory store.

To enable JetSteam for OpenFaaS set `jetstream` as the queue mode in the values.yaml file of the [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md)

```yaml
queueMode: jetstream
nats:
    streamReplication: 1
```

If the NATS pod restarts, you will lose all messages that it contains. In your development or staging environment, this shouldn't happen very often.

For production environments you will need to install NATS separately using its Helm chart with at least 3 server replicas, so that if a pod crashes, the data can be recovered automatically.

```yaml
queueMode: jetstream
nats:
  streamReplication: 3
  external:
    enabled: true
    host: "nats.nats"
    port: "4222"
```

## Features

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

Every OpenFaaS async queue requires a Stream and Consumer to be created on the JetStream server. The queue-worker creates these on first startup if they do not exist. The default Stream and Consumer have some limits:

- The maximum concurrency of each queue is limited to 512.
- There is a limit on the amount of messages that can be queued for retries. NATS will suspend the delivery of messages when this limit is reached. The work will resume once requests or completed or dropped.

To extend these limits Streams and Consumers can be created manually.

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

The Stream will need to be created first. In this example we will create the Stream for the shared queue, `faas-request`.

```bash
nats stream create faas-request \
  --subjects=faas-request \
  --replicas=1 \
  --retention=limits \
  --discard=old
```

Messages intended for a queue are published to a NATS subject, `faas-request` by default and the queue name for a [dedicated queue](/reference/async/#multiple-queues). A Stream should only consume the subject for the queue that it will be associated with. This can be configured with the `--subjects` flag.

The `--replicas` flag is used to configure the stream replication factor. This should be at least 3 for production environments.

The command above includes the required settings for using the Stream with a queue-worker. We want to retain messages based on limits and discard old messages if these limits are reached. You will get prompted interactively for the remaining stream information. Use the defaults or configure your own limits for the stream.

```
? Storage file
? Stream Messages Limit -1
? Per Subject Messages Limit -1
? Total Stream Size -1
? Message TTL -1
? Max Message Size -1
? Duplicate tracking time window 2m0s
? Allow message Roll-ups No
? Allow message deletion Yes
? Allow purging subjects or the entire stream Yes
``` 

#### Create a Consumer

Once the Stream has been created the Consumer can be added. We will create a consumer `faas-workers` for the `faas-request` stream.

```bash
nats consumer \
  create faas-request faas-workers \
  --deliver=all \
  --wait=3m \
  --max-waiting=900 \
  --max-pending=4000
```

This command creates a pull consumer that makes available all messages for every subject on the stream. When you are prompted for the remaining values you can select the defaults.

Important configuration flags:

  - The queue-worker automatically extends the ack window for functions that require more time to complete. In order to prevent us from having to extend the ack window to often we recommend configuring a default acknowledgement waiting time of 3 minutes. This can be configured with the `--wait` flag.

  - The `--max-waiting` flag limits the number of subscribers a queue-worker can create. The value should be at least `max_inflight * queue-worker-replicas`.

    If you are running 3 replicas of the queue-worker with a max_inflight setting of 300 this value should be 900.

  - The `--max-pending` flag limits the number of messages that can have a pending status. Messages that are queued for retries are also considered pending. The value of this flag should be at least `max_inflight * queue-worker-replicas + buffer`. The size of the buffer depends on the number of retries your queue needs to be able to handle. Set this to `-1` to allow any number of pending messages.

    This value can always be update later:

    ```bash
    nats consumer edit --max-pending 6000
    ```

#### Configure the queue-worker
As a final step the queue-worker needs to be configured to use the externally created Stream and Consumer.

For the shared OpenFaaS queue edit the values.yaml file of the OpennFaaS chart. The name of the Consumer used by the queue-worker is set with `jetstreamQueueWorker.durableName`. The name of the Stream needs to be set with `nats.channel`.

```yaml
jetstreamQueueWorker:
  durableName: faas-workers

nats:
  streamReplication: 1
  channel: faas-request
```

A dedicated queue using the [queue-worker Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/queue-worker) can be configured by setting the `nats.stream.name` and `nats.consumer.durableName` parameters.

## See also

- [The Next Generation of Queuing: JetStream for OpenFaaS](https://www.openfaas.com/blog/jetstream-for-openfaas/)
