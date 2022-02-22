## OpenFaaS Pro

OpenFaaS Pro is a commercially licensed distribution of OpenFaaS with additional features, configurations and commercial support from the founders. 

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers, OpenFaaS Pro is meant for production.

### Core platform

Efficiency and redundancy:

* [New auto-scaling engine](/architecture/autoscaling/) to get the scaling just right either on Requests Per Second (RPS), Inflight requests or CPU.
* [Scale idle functions to zero](/openfaas-pro/scale-to-zero) to save on compute costs, increase efficiency and lower your threat profile
* [Retry failed invocations for functions](/openfaas-pro/retries) to handle issues with downstream APIs and back-pressure on concurrency-limited functions

### Workload tuning

In production, it's important to tune functions to make the most of Kubernetes features to increase performance and keep functions healthy.

* Custom HTTP health checks for functions - including path, period seconds and initial delay
* Custom Kubernetes service accounts for functions to access the Kubernetes API
* Custom runtime profiles for security & isolation using gVisor, kata containers etc.
* Custom TerminationGracePeriod for draining work for long running functions

### Events and triggers

Event-driven programming and triggers:

* [Trigger functions via Kafka](/openfaas-pro/kafka-events) for event-driven functions and to integrate with your existing systems
* [Trigger functions from AWS SQS](/openfaas-pro/sqs-events) to integrate with events from AWS.

### Security

* [Single Sign-On using OpenID Connect (OIDC)](/openfaas-pro/sso) means each user authenticates with their own identity, instead of sharing one set of credentials, which is insecure. Use your existing OIDC-compatible Identity Provider (IdP).

### Platform building features

Build functions at scale - for services providers and large teams:

* [Build functions via REST API](/openfaas-pro/builder) using source code without the need to create and manage dozens or hundreds of independent CI jobs.

### On our roadmap

* Scaling upon inflight requests for long running & memory/CPU bound functions (released Jan 2022)
* A new Pro UI dashboard for managing and monitoring OpenFaaS functions across namespaces
* CPU and RAM usage metrics within the OpenFaaS API and Pro UI dashboard
* Concurrency limiting for functions - i.e. one request per container
* Enhanced RBAC for functions and the OpenFaaS REST API
* Integrated support for scaling upon RAM & CPU usage
* AMQP event trigger for RabbitMQ and Azure Service Bus
* Enhanced multi-tenant isolation for large organisations and service providers
* Migration to JetStream from NATS Streaming (which will be deprecated in 2023) 

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
