# Trigger functions from AWS SQS

With our event connector, you can trigger function invocations from messages on AWS SQS queues. This is useful for processing events from AWS services such as S3, CloudWatch and DynamoDB.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

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

    `queueURL` - the URL for your SQS queue

    `visibilityTimeout` - Maximum time to keep message hidden from other processors whilst executing function

    `waitTime` - Time to wait between polling SQS queue for messages.

    `maxMessages` - Maximum messages to fetch at once - between 1-10


## Usage

Once you have configured a number of topics, you can then annotate your functions so that they get triggered by any incoming messages on those topics.

Create a new function:

```bash
export OPENFAAS_PREFIX=ghcr.io/openfaas
faas-cli new --lang go resize-image
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
    lang: go
    handler: ./resize-image
    image: ghcr.io/openfaas:resize-image
```

Test it out:

* Now deploy your function with `faas-cli up`
* Upload an image to your SQS queue
* You'll receive a JSON payload with the details of which file was uploaded
* Fetch the file with the AWS SDK and resize it with a library of your choice
* Finally upload it to a different S3 Bucket.

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
