## OpenFaaS Pro

OpenFaaS Pro is a commercially licensed distribution of OpenFaaS with additional features, configurations and commercial support from the founders. 

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers, OpenFaaS Pro is meant for production.

### Additional capabilities

Eventing:

* [Trigger functions via Kafka](/openfaas-pro/kafka-events) for event-driven functions and to integrate with your existing systems
* [Trigger functions from AWS SQS](/openfaas-pro/sqs-events) to integrate with events from AWS.

Efficiency and redundancy:

* [Scale idle functions to zero](/openfaas-pro/scale-to-zero) to save on cost and increase efficiency
* [Retries for invocations](/openfaas-pro/retries) to handle failures, known problems with downstream APIs and concurrency-limited functions

Security:

* [Single Sign-On using OIDC](/openfaas-pro/sso) for enterprise-grade authentication and more secure credential management

Service providers and large teams:

* [Build functions via REST API](/openfaas-pro/builder) to create your functions from source code, without creating and maintaining hundreds of independent CI jobs.

### On our roadmap

* A new Pro UI dashboard for managing and monitoring OpenFaaS functions across namespaces
* Enhanced RBAC for functions and the OpenFaaS REST API  
* CPU and RAM usage metrics within the OpenFaaS API and Pro UI dashboard
* Scaling upon inflight requests for long running & memory/CPU bound functions
* Integrated support for scaling upon RAM & CPU usage
* Concurrency limiting for functions - i.e. one request per container
* AMQP event trigger for RabbitMQ users and Azure customers
* Enhanced multi-tenant isolation for large organisations and service providers
* Migration to JetStream from NATS Streaming (which is deprecated in 2023) 

Is there something else you need for your organisation? [Get in touch with us here](https://openfaas.com/support/).

### Trusted by

OpenFaaS Pro is trusted by:

* Fortune 500 company (name withheld)
* Fortune 500 company (name withheld)
* Waylay.io
* RateHub.ca
* Check Point Software Technologies Ltd
* Surge "workwithsurge"

Other production users of OpenFaaS include: [ADOPTERS.md](https://github.com/openfaas/faas/blob/master/ADOPTERS.md) 

### Support

OpenFaaS Pro operates on a self-service model with support via email for OpenFaaS Pro features.

OpenFaaS Enterprise comes with a more timely SLA and is suitable for the requirements of larger organisations.

### Getting started

OpenFaaS Pro is for users on Kubernetes, but most of the features are also compatible with faasd. Customers can request configuration for OpenFaaS Pro for faasd via support.

Are you interested in OpenFaaS for your organisation? [Contact us](https://openfaas.com/support/) to find out more.
