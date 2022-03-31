## OpenFaaS Pro

OpenFaaS Pro is a commercially licensed distribution of OpenFaaS with additional features, configurations and commercial support from the founders. 

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers, OpenFaaS Pro is meant for production.

### Core platform

Efficiency and redundancy:

* [New auto-scaling engine](/architecture/autoscaling/) to get the scaling just right either on Requests Per Second (RPS), Inflight requests or CPU.
* [Scale idle functions to zero](/openfaas-pro/scale-to-zero) to save on compute costs, increase efficiency and lower your threat profile
* [Retry failed invocations for functions](/openfaas-pro/retries) to handle issues with downstream APIs and back-pressure on concurrency-limited functions
* CPU and RAM usage metrics for every function to measure usage and fine-tune limits

[![Scaling that works for various types of functions](https://pbs.twimg.com/media/FJ9EBVdWQAM9DeW?format=jpg&name=medium)](/architecture/autoscaling/)
> Scaling that works for various types of functions

The OpenFaaS Pro autoscaler emits detailed usage information, meaning functions can scale not just on requests per second, but inflight requests and CPU utilisation.

Usability and operations:

[![The OpenFaaS dashboard](https://pbs.twimg.com/media/FPKw2VUWYA4pzV7?format=jpg&name=medium)](https://twitter.com/alexellisuk/status/1509463370088521728/)

The OpenFaaS dashboard integrates with CPU & RAM usage metrics, and container logs to give you insights on your functions in one place. You can also add metadata from your source control management tool like a SHA, owner, project or URL to the source code.

### Events and triggers

Event-driven programming and triggers:

* [Trigger functions via Kafka](/openfaas-pro/kafka-events) for event-driven functions and to integrate with your existing systems
* [Trigger functions from AWS SQS](/openfaas-pro/sqs-events) to integrate with events from AWS.

### Workload tuning

To avoid errors when scaling up or down, you may need to tune your function's configuration to suit how it works. Pro users get access to fine-tune health checks, set custom Kubernetes service accounts and termination grace periods for shutting down functions.

* Custom HTTP health checks for functions - including path, period seconds and initial delay
* Custom Kubernetes service accounts for functions to access the Kubernetes API
* Custom runtime profiles for security & isolation using gVisor, kata containers etc.
* Custom TerminationGracePeriod for draining work for long running functions

Read more: [OpenFaaS workloads](https://docs.openfaas.com/reference/workloads/) and [Custom Profiles](https://docs.openfaas.com/reference/profiles/#use-an-alternative-runtimeclass)

### Enterprise security

* [Single Sign-On using OpenID Connect (OIDC)](/openfaas-pro/sso) means each user authenticates with their own identity, instead of sharing one set of credentials, which is insecure. Use your existing OIDC-compatible Identity Provider (IdP).

### Platform building features

Build functions at scale - for services providers and large teams:

* [Build functions via REST API](/openfaas-pro/builder) using source code without the need to create and manage dozens or hundreds of independent CI jobs.

### On our roadmap

* Scaling upon inflight requests for long running & memory/CPU bound functions (released Jan 2022)
* A new Pro UI dashboard for managing and monitoring OpenFaaS functions across namespaces (released March 2022)
* CPU and RAM usage metrics within the OpenFaaS API, CLI and Pro UI dashboard (released Feb 2022)
* Migration to JetStream from NATS Streaming (which will be deprecated in 2023) 
* Concurrency limiting for functions - i.e. one request per container
* Enhanced RBAC for functions and the OpenFaaS REST API
* AMQP event trigger for RabbitMQ and Azure Service Bus
* Enhanced multi-tenant isolation for large organisations and service providers

Is there something else you need for your team or organisation? [Get in touch with us here](https://openfaas.com/support/).

### Trusted by

OpenFaaS Pro is trusted by:

* Fortune 500 company (name withheld)
* Fortune 500 company (name withheld)
* Waylay.io
* RateHub.ca
* Check Point Software Technologies Ltd
* Surge "workwithsurge"

Tell us about your use-case for OpenFaaS and see what other companies are doing in the: [ADOPTERS.md file](https://github.com/openfaas/faas/blob/master/ADOPTERS.md) 

### Support

OpenFaaS Pro operates on a self-service model with support via email for OpenFaaS Pro features.

OpenFaaS Enterprise comes with a more timely SLA and is suitable for the requirements of critical applications or larger organisations.

### Getting started

OpenFaaS Pro is for users on Kubernetes, but most of the features are also compatible with faasd. Customers can request configuration for OpenFaaS Pro for faasd via support.

Are you interested in OpenFaaS for your organisation? [Contact us](https://openfaas.com/support/) to find out more.
