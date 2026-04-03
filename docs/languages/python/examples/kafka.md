Publish messages to a Kafka topic from a function using the `confluent-kafka` package. This lets you bridge HTTP-triggered functions into event-driven pipelines, using Kafka as a decoupling layer between your API and downstream consumers.

Use-cases:

* Publishing events or audit logs to a Kafka topic
* Decoupling workloads by writing to a message bus
* Feeding data pipelines from HTTP endpoints

This example uses the `confluent-kafka` package with SASL/SSL authentication. Broker credentials are stored as [OpenFaaS secrets](/reference/secrets/).

If you'd like to trigger functions from Kafka topics instead, see [Trigger functions from Kafka](/openfaas-pro/kafka-events).

## Overview

handler.py:

```python
import os
import socket
from confluent_kafka import Producer

# Initialise the producer once and reuse it across invocations
# to keep the broker connection alive between requests.
kafkaProducer = None

def initProducer():
    username = read_secret('kafka-broker-username')
    password = read_secret('kafka-broker-password')
    broker = os.getenv("kafka_broker")

    conf = {
        'bootstrap.servers': broker,
        'security.protocol': 'SASL_SSL',
        'sasl.mechanism': 'PLAIN',
        'sasl.username': username,
        'sasl.password': password,
        'client.id': socket.gethostname()
    }

    return Producer(conf)

def handle(event, context):
    global kafkaProducer

    if kafkaProducer is None:
        kafkaProducer = initProducer()

    topic = 'faas-request'

    # Produce the request body as a message and wait for delivery
    kafkaProducer.produce(topic, value=event.body)
    kafkaProducer.flush()

    return {
        "statusCode": 200,
        "body": "Message produced to {}".format(topic)
    }

def read_secret(name):
    with open("/var/openfaas/secrets/" + name, "r") as f:
        return f.read().strip()
```

requirements.txt:

```
confluent-kafka
```

stack.yaml:

```yaml
functions:
  kafka-producer:
    lang: python3-http-debian
    handler: ./kafka-producer
    image: ttl.sh/openfaas-examples/kafka-producer:latest
    environment:
      kafka_broker: "<your-broker-endpoint>:9092"
    secrets:
    - kafka-broker-username
    - kafka-broker-password
```

The Debian variant of the template is required because `confluent-kafka` depends on `librdkafka`, a native C library that will not build on Alpine.

The Kafka producer is initialised once on first invocation and reused for subsequent requests, keeping the broker connection alive between calls and avoiding the overhead of re-authenticating on every request.

The `SASL_SSL` security protocol combines SASL authentication with TLS encryption. The `sasl.mechanism` must match your broker's configuration:

- `PLAIN` — standard for managed services such as Confluent Cloud and Aiven.
- `SCRAM-SHA-256` / `SCRAM-SHA-512` — common for self-hosted brokers.
- `GSSAPI` — Kerberos-based authentication.

## Step-by-step walkthrough

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-http-debian
faas-cli new --lang python3-http-debian kafka-producer \
  --prefix ttl.sh/openfaas-examples
```

The example uses the public [ttl.sh](https://ttl.sh) registry — replace the prefix with your own registry for production use.

Update `kafka-producer/handler.py` and `kafka-producer/requirements.txt` with the code from the overview above.

### Create secrets for Kafka broker credentials

Store your Kafka broker username and password as OpenFaaS secrets. This keeps credentials out of environment variables and the function's container image.

Save your broker username to `kafka-broker-username.txt` and your broker password to `kafka-broker-password.txt`, then run:

```bash
faas-cli secret create kafka-broker-username --from-file kafka-broker-username.txt
faas-cli secret create kafka-broker-password --from-file kafka-broker-password.txt
```

At runtime, the secrets are mounted as files under `/var/openfaas/secrets/` inside the function container.

### Deploy and invoke

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter kafka-producer \
 --tag digest
```

Publish a message to the Kafka topic by invoking the function:

```bash
curl http://127.0.0.1:8080/function/kafka-producer \
  --data "Hello from OpenFaaS"
```
