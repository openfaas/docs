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

[![The OpenFaaS dashboard](https://pbs.twimg.com/media/FPKw2VUWYA4pzV7?format=jpg&name=medium)](/openfaas-pro/dashboard/)

The OpenFaaS dashboard integrates with CPU & RAM usage metrics, and container logs to give you insights on your functions in one place. You can also add metadata from your source control management tool like a SHA, owner, project or URL to the source code.

* [OpenFaaS Dashboard](/openfaas-pro/dashboard/)

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

### Comparison

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers, OpenFaaS Pro is meant for production.

Support

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Suitability           | Open Source developers and basic exploration  | Production, business critical, or PoC | Production, business critical, or PoC |
| Support via email     | N/a               | Pro features only      | OSS & Pro within 1 business day |
| Support via GitHub    | N/a               | Pro features only      | OSS & Pro within 1 business day |
| Support via Slack     | N/a               | N/a                    | Up to 5 developers              |
| Private customers' community   | N/a               | Private access to GitHub community for two people | Private access to GitHub community for team |  

Features

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Dashboard         | Basic, legacy UI portal (in code-freeze)  | Dashboard with metrics, logs and multiple namespace support | As per Pro |
| Metrics         | Basic HTTP invocation metrics  | Plus advanced CPU/RAM usage metrics      | As per Pro |
| Autoscaling granularity   | Single rule for all functions | Custom per functions      | As per Pro |
| Autoscaling strategy   | RPS-only | CPU utilization, Capacity (inflight requests) or RPS      | As per Pro |
| Authentication | Shared token with every user | Sign-On with OIDC Okta/Auth0 | Custom Single Sign-On with your IdP |
| Scale to Zero | Not supported | Custom rule per function | As per Pro |
| Custom Kubernetes service account      | N/a             | Supported per function | As per Pro |

Durability and reliability

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Health checks | Not supported | Custom HTTP path and intervals per function | As per Pro |
| Retry failed invocations | Not supported | Retry certain HTTP codes with a back-off | As per Pro |
| GDPR                  | Sensitive data printed in logs | Sensitive data omitted from logs | As per Pro | 
| Grafana Dashboard      | N/a             | Supplied with advanced metrics in private repository | As per Pro |
| Secure isolation with Kata containers or gVisor      | N/a             | Supported via a runtimeClass | As per Pro |

Event-brokers

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Kafka broker | Not supported | Supports SASL or TLS auth, Aiven, Confluent and self-hosted | As per Pro |
| AWS SQS | Not supported | Supported | As per Pro |

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
