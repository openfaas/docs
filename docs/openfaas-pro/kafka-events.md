# Trigger functions from Kafka

Trigger function invocations from messages received on Kafka topics.

> Note: This feature is included for [OpenFaaS Standard & For Enterprises](https://openfaas.com/pricing/) customers.

## Feature blog posts

* [Trigger your functions from Kafka with Confluent Cloud](https://www.openfaas.com/blog/confluent-kafka/)
* [Event-driven OpenFaaS with Managed Kafka from Aiven](https://www.openfaas.com/blog/openfaas-kafka-aiven/)

## Walkthrough video

<iframe width="560" height="315" src="https://www.youtube.com/embed/jUFizTM3iKw?si=JMxPXOywXocaP4-m" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Installation

You can install the Kafka connector using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/kafka-connector), or by using arkade.

### Installation with Helm

See the [kafka-connector helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/kafka-connector)

## Usage

The topics you need to connect to functions are set on a connector.

A single connector can have either a single topic, or a comma-separated list of topics.

It is recommended to deploy one connector per topic, so that it can be scaled to match the number of consumers set on the partition. I.e. a topic named `payment.created` which has a partition size of 3, should have three replicas of the configured connector deployed.

But it's also possible to use a single connector and pass in multiple topics i.e. `payment.created,customer.onboarded,invoice.generate`.

Your function(s) can then subscribe to one or more topics by setting the topic annotation with the name of the topic, or a comma separated list of topics.

Create a new function:

```bash
export OPENFAAS_PREFIX=docker.io/alexellis2
faas-cli new --lang golang-middleware provision-customer
```

Now add an annotation for the `payment.created` topic, so that the `provision-customer` function is invoked for any message received:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  provision-customer:
    annotations:
      topic: payment.created
    lang: golang-middleware
    handler: ./provision-customer
    image: docker.io/alexellis2/provision-customer:latest
```

Now deploy your function, and publish an event to your Kafka broker on the `payment.created` topic.

### Message body, headers and metadata

The body of the message in binary format will be received as the body of the function or incoming HTTP request.

For headers and metadata:

* `X-Topic` - the Kafka topic
* `X-Kafka-*` - each header on the Kafka message is outputted with the form: `X-Kafka-Key: Value`
* `X-Kafka-Partition` - the partition in Kafka that the message was received on
* `X-Kafka-Offset` - the offset in Kafka that the message was received on
* `X-Kafka-Key` - if set on the message, the key of the message in Kafka

If a binary message key is submitted to a Kafka topic, then the value will be base64-encoded and passed to the function as the `X-Kafka-Key` header, along with an additional header `X-Kafka-Key-Enc` with a value of `b64` to indicate that the value is base64-encoded.

The connector will invoke functions using the default content type of `text/plain`, however [this can be overridden in the Helm chart](https://github.com/openfaas/faas-netes/blob/master/chart/kafka-connector/values.yaml) to i.e. `application/json or whatever is required.

Most templates make HTTP headers sent by the connector available through their request or context object, for example:

* [golang-middleware](/languages/go/)
* [python3-http](/languages/python/)
* [node22](/languages/node/)

For detailed examples with Node.js, see: [Serverless For Everyone Else](http://store.openfaas.com/l/serverless-for-everyone-else)

For detailed examples with Go, see: [Everyday Golang (Premium Edition only)](https://openfaas.gumroad.com/l/everyday-golang)

### Message lifecycle and retries

**Default synchronous invocation**

By default, the Kafka connector will invoke functions using the Gateway's synchronous invocation endpoint. If a response is returned by a function, then the message will be considered as processed and will be acknowledged on the topic.

**Automatic retries via the queue-worker**

If you would like to retry messages then you can switch the connector to use asynchronous invocations. Asynchronous invocations are executed using the queue-worker.

The queue-worker has a set of default retry values set via the Helm chart. They can also be overridden for each function using the documentation on the [Retries page](/openfaas-pro/retries).

## See also

* [Event-driven OpenFaaS with Managed Kafka from Aiven](https://www.openfaas.com/blog/openfaas-kafka-aiven/)
* [Quick start with the Helm chart - self-hosted, SASL and TLS Client Certificates](https://github.com/openfaas/faas-netes/blob/master/chart/kafka-connector/quickstart.md)
* Original launch blog post: [Staying on topic: trigger your OpenFaaS functions with Apache Kafka](https://www.openfaas.com/blog/kafka-connector/)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Talk to us](https://openfaas.com/pricing/)
