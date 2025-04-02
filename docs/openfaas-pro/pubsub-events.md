# Trigger function from Google Cloud Pub/Sub events

Trigger OpenFaaS functions from Google Cloud Pub/Sub messages.

> Note: This feature is included for [OpenFaaS Standard & For Enterprises](https://openfaas.com/pricing/)

## Installation

* Setup the connector

    You can install the Google Cloud Pub/Sub connector using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/gcp-pubsub-connector).

    The `values.yaml` file can be customised to suit your needs.

*  Configure the connector for your needs by defining a values.yaml file

    ```yaml
    # Google cloud project ID
    projectID: "openfaas-381517"

    # List if Pub/Sub subscriptions the connector should subscribe to.
    subscriptions:
      - my-sub
    ```

Use the `subscriptions` parameter to configure a list of Pub/Sub subscriptions to which the connector should subscribe. When the subscriber receives a message, the connector will attempt to invoke any function that has the subscription name listed in its `topic` annotation.

## Usage

To test the connector, you can deploy the [printer function](https://github.com/openfaas/store-functions/blob/master/printer/handler.go). It prints out the HTTP headers and body of any invocation.

Add an annotation `topic=sub1` so that the function is invoked for any message delivered to the `sub1` Pub/Sub subscription.

Each time a function is invoked by the connector it will receive the message body as the HTTP body. Message attributes are mapped to HTTP Headers.

```bash
faas-cli store deploy printer \
  --annotation topic=sub1
```

Use the [Google Cloud Console](https://console.cloud.google.com) or [gcloud CLI](https://cloud.google.com/pubsub/docs/publish-receive-messages-gcloud) to publish a message for testing.

```bash
faas-cli logs printer

2025-04-02T11:40:50Z X-Pubsub-Attr-Foo=[Y]
2025-04-02T11:40:50Z X-Pubsub-Msg-Id=[14373415254515458]
2025-04-02T11:40:50Z User-Agent=[openfaas-gateway/0.4.39]
2025-04-02T11:40:50Z Accept-Encoding=[gzip]
2025-04-02T11:40:50Z X-Forwarded-For=[127.0.0.1:49964]
2025-04-02T11:40:50Z X-Topic=[my-sub]
2025-04-02T11:40:50Z X-Forwarded-Host=[127.0.0.1:8080]
2025-04-02T11:40:50Z X-Pubsub-Publish-Time=[2025-04-02T11:40:49Z]
2025-04-02T11:40:50Z X-Start-Time=[1743594050512199246]
2025-04-02T11:40:50Z Content-Type=[text/plain]
2025-04-02T11:40:50Z X-Call-Id=[f7fccd26-d8ee-4874-90e3-eefc94e2b74c]
2025-04-02T11:40:50Z X-Connector=[connector-sdk openfaasltd/gcp-pubsub-connector]
2025-04-02T11:40:50Z 
2025-04-02T11:40:50Z Hello OpenFaaS
```

Additional headers are made available to the request. These headers contain any message attributes along with some metadata about the Pub/Sub message.

* `X-PubSub-Msg-ID` - the Pub/Sub message identifier.
* `X-PubSub-Publish-Time` - the time at which the message was published.
* `X-PubSub-Delivery-Attempt` - the number of times a message has been delivered.
* `X-PubSub-Ordering-Key` - identifies related messages for which publish order should be respected.
* `X-PubSub-Attr-<key>` - value of Pub/Sub message attribute.

So for example, if you added an attributed called `Foo` with a value of `Y`, you'd get the following extra header: `X-PubSub-Attr-Foo: Y`

The default content-type is configured as `text/plain`, but can be changed to another content-type such as `application/json` or `application/octet-stream` by the [values.yaml file](https://github.com/openfaas/faas-netes/blob/master/chart/gcp-pubsub-connector/values.yaml) for the connector.

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Talk to us](https://openfaas.com/pricing/)