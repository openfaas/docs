# Trigger functions from Kafka

Trigger function invocations from messages received on Kafka topics.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

## Installation

You can install the Kafka connector using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/kafka-connector), or by using arkade.

Are you an Aiven customer? See also: [Event-driven OpenFaaS with Managed Kafka from Aiven](https://www.openfaas.com/blog/openfaas-kafka-aiven/)

### Installation with arkade

```bash
export TOPICS="payment.created"
arkade install kafka-connector \
 --broker-host kafka-broker \
 --topics $TOPICS \
 --license-file $HOME/.openfaas/LICENSE
```

### Installation with Helm

See [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/kafka-connector)

## Usage

The topics you need to connect to functions are set on a connector.

A single connector can have either a single topic, or a comma-separated list of topics.

Customers tend to prefer to deploy a single copy of the connector for each topic, so that they can be scaled to match the number of consumers set on the partition.

But it's also possible to use a single connector and pass in multiple topics i.e. `payment.created,customer.onboarded,invoice.generate`.

Your function(s) can then subscribe to one or more topics by setting the topic annotation with the name of the topic, or a comma separated list of topics.

Create a new function:

```bash
export OPENFAAS_PREFIX=ghcr.io/openfaas
faas-cli new --lang go provision-customer
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
    lang: go
    handler: ./provision-customer
    image: ghcr.io/openfaas:provision-customer
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

The default content-type is configured as `text/plain`, but can be changed to another content-type such as `application/json` or `application/octet-stream` by the [values.yaml file](https://github.com/openfaas/faas-netes/blob/master/chart/kafka-connector/values.yaml) for the connector.

Most templates make these variables available through their request or context object, for example:

* [golang-middleware](https://github.com/openfaas/golang-http-template)
* [python3-http](https://github.com/openfaas/python-flask-template)
* [node17](https://docs.openfaas.com/cli/templates/#nodejs-templates-of-watchdog-template)

For detailed examples with Node.js, see: [Serverless For Everyone Else](https://gumroad.com/l/serverless-for-everyone-else)

For detailed examples with Go, see: [Everyday Golang (Premium Edition)](https://openfaas.gumroad.com/l/everyday-golang)

## See also

* [Event-driven OpenFaaS with Managed Kafka from Aiven](https://www.openfaas.com/blog/openfaas-kafka-aiven/)
* [Quick start with the Helm chart - self-hosted, SASL and TLS Client Certificates](https://github.com/openfaas/faas-netes/blob/master/chart/kafka-connector/quickstart.md)
* Launch blog post: [Staying on topic: trigger your OpenFaaS functions with Apache Kafka](https://www.openfaas.com/blog/kafka-connector/)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
