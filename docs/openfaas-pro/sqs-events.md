# Trigger functions from AWS SQS

With our event connector, you can trigger function invocations from messages on AWS SQS queues. This is useful for processing events from AWS services such as S3, CloudWatch and DynamoDB.

> Note: This feature is included for [OpenFaaS Standard & For Enterprises](https://openfaas.com/pricing/) customers.

On the blog: [How to integrate OpenFaaS functions with managed AWS services](https://www.openfaas.com/blog/integrate-openfaas-with-managed-aws-services/)

## Installation

### Pre-reqs on AWS

* Create an SQS queue
* Create an IAM user with permissions to write to your SQS queue
* Create an S3 bucket
* Configure your S3 bucket to publish events to your SQS queue

### On Kubernetes

* Set up AWS

    You can configure permissions using a dedicated IAM user, or if your cluster is configured for AWS IAM, ambient credentials mapped into the Pod at runtime.

* Set up the connector

    You can install the SQS connector using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/sqs-connector).

    The values.yaml file can be customised to suit the needs of your SQS queue and the consumer.

* Tuning the connector for your needs

    `queueURL` - the URL for your SQS queue - you can specify a comma-separated list of queues to consume from or a single URL of a queue. When you specify multiple queues, the connector will consume from each of them and invoke functions in parallel. The access key and secret key must be the same for all queues given in `queueURL`.

    `visibilityTimeout` - Maximum time to keep message hidden from other processors whilst executing function

    `waitTime` - Time to wait between polling SQS queue for messages.

    `maxMessages` - Maximum messages to fetch at once - between 1-10

Each time a function is invoked by the connector it will receive the message from the queue as the HTTP body.

It'll also receive two HTTP headers:

* `X-SQS-Message-ID`- the ID of the message in the SQS queue for when a function needs to do something with the message
* `X-SQS-Queue-URL` - the URL of the SQS queue - required when a function receives messages from multiple queues

Once the message has been delivered to the function, it will be deleted from the queue.

## Usage

Once you have configured a number of topics, you can then annotate your functions so that they get triggered by any incoming messages on those topics.

Download a template such as the `golang-middleware` template:

```bash
faas-cli template store pull golang-middleware
```

Then scaffold a new function using your registry in the `OPENFAAS_PREFIX` environment variable:

```bash
export OPENFAAS_PREFIX=ghcr.io/openfaas
faas-cli new --lang golang-middleware resize-image
```

Now add an annotation for the `s3-put-image` queue, so that the `resize-image` function is invoked for any message received:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  resize-image:
    annotations:
      topic: s3-put-image
    lang: golang-middleware
    handler: ./resize-image
    image: ghcr.io/openfaas/resize-image:latest
```

Edit the HTTP handler at `./s3-put-image/handler.go` so it prints out the HTTP body and headers to its logs.

You can find a complete example here: [printer function written in Go](https://github.com/openfaas/store-functions/tree/master/printer).

Test it out:

* Now deploy your function with `faas-cli up`
* Upload an image to your SQS queue
* You'll receive a JSON payload with the details of which file was uploaded
* Fetch the file with the AWS SDK and resize it with a library of your choice
* Upload it to a different S3 Bucket.
* Now view the logs for your function to see the output

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Talk to us](https://openfaas.com/pricing/)
