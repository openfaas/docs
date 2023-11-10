# Trigger functions from AWS SNS messages

With the SNS connector AWS SNS subscriptions can be used to trigger OpenFaaS functions from many different kinds of AWS services such as S3, DynamoDB, Redshift and so on.

> Note: This feature is included for [OpenFaaS Standard & For Enterprises](https://openfaas.com/pricing/) customers.

On the blog: [How to integrate OpenFaaS functions with managed AWS services](https://www.openfaas.com/blog/integrate-openfaas-with-managed-aws-services/)

## Installation

### Pre-reqs on AWS

* Create a SNS topic
    The connector can only be used with topics Standard topics and not FIFO topics. Make sure your topic uses the right type.

* Take note of the topic ARN


### Install the connector with Helm

* Set up AWS
    Create the credentials in the AWS Dashboard. You can configure permissions using a dedicated IAM user, or if your cluster is configured for AWS IAM, ambient credentials mapped into the Pod at runtime.

* Deploy the connector with Helm
    The SNS connector can be installed using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/sns-connector)


* Configure the connector for you needs by defining a values.yaml file
        
    ```yaml

    # Public callback URL for subscriptions
    callbackURL: "http://sns.example.com/callback"

    # SNS topic ARN
    topicARN: "arn:aws:sns:us-east-1:123456789012:of-event"

    # AWS shared credentials file:
    awsCredentialsSecret: aws-sns-credentials

    awsRegion: us-east-1
    ```
> Note that the callback URL has to be publicly accessible to receive SNS messages. An ingress configuration example is available in the [chart README](https://github.com/openfaas/faas-netes/tree/master/chart/sns-connector#configure-ingress)

## Usage

Once you configured the connector to receive callbacks for an SNS topic you can annotate you functions so they get triggered by any incoming message on that topic.

```bash
faas-cli store deploy printer \
  --annotation topic=arn:aws:sns:us-east-1:222943518635:of-event
```

The printer function will print out the body and headers of each request which makes it a great function for debugging events.

You can publish a message to your topic from the AWS console. 

Example output of the printer function:

```bash
faas-cli logs printer

2023-01-13T15:05:46Z User-Agent=[Go-http-client/1.1]
2023-01-13T15:05:46Z Accept-Encoding=[gzip]
2023-01-13T15:05:46Z 2023/01/13 15:05:46 POST / - 202 Accepted - ContentLength: 0B (0.0004s)
2023-01-13T15:05:46Z X-Topic=[arn:aws:sns:us-east-1:222943518635:of-event]
2023-01-13T15:05:46Z X-Call-Id=[c003629d-81dc-4d7a-bacd-96bb2a29782c]
2023-01-13T15:05:46Z X-Sns-Attr-Count=[20]
2023-01-13T15:05:46Z X-Sns-Attr-Foo-Type=[String]
2023-01-13T15:05:46Z X-Connector=[connector-sdk openfaasltd/sns-connector]
2023-01-13T15:05:46Z X-Sns-Arn=[arn:aws:sns:us-east-1:222943518635:of-event]
2023-01-13T15:05:46Z X-Sns-Attr-Foo=[bar]
2023-01-13T15:05:46Z X-Start-Time=[1673622346748456857]
2023-01-13T15:05:46Z X-Sns-Attr-Count-Type=[Number]
2023-01-13T15:05:46Z X-Sns-Message-Id=[6f0ee500-756a-5391-9742-b93b00e0afb0]
2023-01-13T15:05:46Z X-Sns-Subject=[openfaas]
2023-01-13T15:05:46Z 
2023-01-13T15:05:46Z Hello OpenFaaS
```

Additional headers are made available on the request. These headers contain some SNS message metadata and the message attributes.

* `X-Sns-Message-Id` - the message id.
* `X-Sns-Subject` - the message subject.
* `X-Sns-Arn` - the arn of the SNS topic.
* `X-Sns-Attr-<key>` - value of an SNS message attribute.
* `X-Sns-Attr-<Key>-Type` - type of the SNS message attribute.

So for example, if you added an attributed called X with a value of Y and a type of `String`, you'd get the following two extra headers:

* `X-Sns-Attr-X: Y`
* `X-Sns-Attr-X-Type: String`

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Talk to us](https://openfaas.com/pricing/)