# Trigger functions from RabbitMQ

Trigger OpenFaaS functions from RabbitMQ messages.

> Note: This feature is included for [OpenFaaS Standard & For Enterprises](https://openfaas.com/pricing/) customers.

## Installation

* Setup the connector

    You can install the RabbitMQ connector using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/rabbitmq-connector).

    The `values.yaml` file can be customised to suit your needs.

*  Configure the connector for you needs by defining a values.yaml file

    ```yaml
    rabbitmqURL: "amqp://rabbitmq.rabbitmq.svc.cluster.local:5672"

    queues:
      - name: queue1
        durable: true
    ```

Use the `queues` parameter to configure a list of queues to which the connector should subscribe. When a message is received on a queue, the connector will invoke any function that has the queue name listed in its `topic` annotation.

Queue configuration:

* `name` - The name of the queue. (Required)
* `durable` - Specifies whether the queue should survive broker restarts.
* `nowait` - Whether to wait for confirmation from the server when declaring the queue.
* `autodelete` - Whether the queue should be deleted automatically when no consumers are connected.
* `exclusive` - Specifies whether the queue should be exclusive to the connection that created it.

If your RabbitMQ requires a TLS connection make sure to use `amqps://` as the scheme in the url.

## Usage

To test the connector, you can deploy the [printer function](https://github.com/openfaas/store-functions/blob/master/printer/handler.go). It prints out the HTTP headers and body of any invocation.

Add an annotation `topic=queue1` so that the function is invoked for any message published to the queue with name queue1.
Each time a function is invoked by the connector it will receive the message from the queue as the HTTP body.

```bash
faas-cli store deploy printer \
  --annotation topic=queue1
```

Publish a message on the queue for testing. We will be using the [RabbitMQ management CLI](https://www.rabbitmq.com/docs/management-cli) for this.

```bash
./rabbitmqadmin publish \
  routing_key="queue1" payload='Hello, Task Queue!' properties='{"message_id":"1"}'
```

Take a look at the logs of the printer function to inspect the invocations made by the connector. You should see the following output:

```bash
faas-cli logs printer

2024-12-03T14:36:03Z X-Connector=[connector-sdk openfaasltd/rabbitmq-connector]
2024-12-03T14:36:03Z X-Topic=[queue1]
2024-12-03T14:36:03Z Accept-Encoding=[gzip]
2024-12-03T14:36:03Z Content-Type=[text/plain]
2024-12-03T14:36:03Z X-Call-Id=[d0ab9f9e-0c93-46b1-a4dd-695a037acb38]
2024-12-03T14:36:03Z 2024/12/03 14:36:03 POST / - 202 Accepted - ContentLength: 0B (0.0003s)
2024-12-03T14:36:03Z X-Forwarded-Host=[gateway.openfaas:8080]
2024-12-03T14:36:03Z X-Rabbitmq-Msg-Id=[1]
2024-12-03T14:36:03Z X-Start-Time=[1733236563468386644]
2024-12-03T14:36:03Z X-Forwarded-For=[10.42.0.13:55796]
2024-12-03T14:36:03Z X-Rabbitmq-Routing-Key=[queue1]
2024-12-03T14:36:03Z User-Agent=[openfaas-gateway/0.4.34]
2024-12-03T14:36:03Z 
2024-12-03T14:36:03Z Hello, Task Queue!
2024-12-03T14:36:03Z 
```

Additional headers are made available to the request. These headers contain RabbitMQ message metadata.

* `X-Topic` - topic that triggered the function.
* `X-Rabbitmq-Msg-Id` - the message identifier.
* `X-Rabbitmq-Routing-Key` - the routing key of the message.

The default content-type is configured as `text/plain`, but can be changed to another content-type such as `application/json` or `application/octet-stream` by the [values.yaml file](https://github.com/openfaas/faas-netes/blob/master/chart/kafka-connector/values.yaml) for the connector.

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Talk to us](https://openfaas.com/pricing/)
