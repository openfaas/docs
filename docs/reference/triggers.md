# Triggers

OpenFaaS functions can be triggered easily by any kind of event. A small piece of code will convert from the event-source and trigger the function using the OpenFaaS Gateway API.

The most common use-case is HTTP which acts as a lingua franca between internet-connected systems.

Looking to trigger a function on a schedule? Have a look at the [Cron page](/reference/cron/) for more information

## HTTP
This is the default, and standard method for interacting with your Functions.

The function URL follows the pattern of:
```
https://<gateway URL>:<port>/function/<function name>
```

> There is also the ability to execute a function asynchronously by replacing `/function/` with `/async-function/` before the function name

## CLI
Trigger a function using the `faas-cli` by using the function name

```
echo "triggered" | faas-cli invoke figlet
```

> CLI invocation can also be async by passing the `-a` flag to the `invoke` call

### Other Event Sources

#### Kafka
Connect your function(s) to Kafka topics

More information in the Incubator repository: [openfaas-incubator/kafka-connector](https://github.com/openfaas-incubator/kafka-connector)

#### AWS SNS
Trigger a function from AWS SNS Notifications and Subscriptions

More information in the repository: [affix/OpenFaaS-SNS](https://github.com/affix/OpenFaaS-SNS)

#### CloudEvents
CloudEvents is a specification for describing event data in a common way. More information on [CloudEvents](https://cloudevents.io/)

Trigger functions from Azure EventGrid with the CloudEvents standard

More information in the repository: [johnmccabe/cloudevents-slack-demo](https://github.com/johnmccabe/cloudevents-slack-demo)

#### RabbitMQ
Invoke functions from RabbitMQ topics

More information in the repository: [Templum/rabbitmq-connector](https://github.com/Templum/rabbitmq-connector)
