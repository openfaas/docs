# Trigger functions from Kafka

Trigger function invocations from messages received on Kafka topics.

> Note: This functionality is part of [OpenFaaS Pro](https://openfaas.com/support/).

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

Once you have configured a number of topics, you can then annotate your functions so that they get triggered by any incoming messages on those topics.

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

## See also

* [Event-driven OpenFaaS with Managed Kafka from Aiven](https://www.openfaas.com/blog/openfaas-kafka-aiven/)
* [Quick start with the Helm chart - self-hosted, SASL and TLS Client Certificates](https://github.com/openfaas/faas-netes/blob/master/chart/kafka-connector/quickstart.md)
* Launch blog post: [Staying on topic: trigger your OpenFaaS functions with Apache Kafka](https://www.openfaas.com/blog/kafka-connector/)
