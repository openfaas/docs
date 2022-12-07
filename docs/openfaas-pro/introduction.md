## OpenFaaS Pro

OpenFaaS Pro is a commercially licensed distribution of OpenFaaS with additional features, configurations and commercial support from the founders. 

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers and basic exploration, OpenFaaS Pro is meant for production.

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
* Custom support for probing Istio endpoints during scale from zero

Read more: [OpenFaaS workloads](https://docs.openfaas.com/reference/workloads/) and [Custom Profiles](https://docs.openfaas.com/reference/profiles/#use-an-alternative-runtimeclass)

### Enterprise security

* [Single Sign-On using OpenID Connect (OIDC)](/openfaas-pro/sso) means each user authenticates with their own identity, instead of sharing one set of credentials, which is insecure. Use your existing OIDC-compatible Identity Provider (IdP).

### Platform building features

Build functions at scale - for services providers and large teams:

* [Build functions via REST API](/openfaas-pro/builder) using source code without the need to create and manage dozens or hundreds of independent CI jobs.

### On our roadmap

Recently released:

* Scaling upon inflight requests for long running & memory/CPU bound functions (released Jan 2022)
* A new Pro UI dashboard for managing and monitoring OpenFaaS functions across namespaces (released March 2022)
* CPU and RAM usage metrics within the OpenFaaS API, CLI and Pro UI dashboard (released Feb 2022)
* AWS SQS event-connector (released January 2022)
* Concurrency limiting for functions - i.e. one request per container
* Redesigned async system with NATS JetStream which replaces NATS Streaming (available in pilot)
    * NATS Streaming is available for CE and will be deprecated in June 2023.
    * Dedicated Helm chart for installing additional queue-workers with JetStream
    * Structured logging / JSON for OpenFaaS Pro customers in the queue-worker
* Postgres event connector using either WAL or LISTEN/NOTIFY

Upcoming:

* Enhanced RBAC for functions and the OpenFaaS REST API
* AMQP event trigger for RabbitMQ and Azure Service Bus
* Enhanced multi-tenant isolation for large organisations and service providers

Is there something else you need for your team or organisation? [Get in touch with us here](https://openfaas.com/support/).

### Comparison

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers and basic exploration, OpenFaaS Pro is meant for production.

OpenFaaS Pro is a distribution of OpenFaaS with additional features and configurations that we believe customers need to operate a product or service in production.

As we tune OpenFaaS Pro for our existing customers, we improve it for everyone else at the same time - writing articles, tuning configuration and adding key features.

We see it as the start of a two-way relationship and an opportunity to collaborate directly with our team.

**Support**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Suitability           | Open Source developers and basic exploration  | Production, business critical, or PoC | Production, business critical, or PoC |
| Support via email     | N/a               | Pro features only      | Response within 1 business day |
| Support via GitHub    | N/a               | Pro features only      | Response within 1 business day |
| Support via Slack     | N/a               | N/a                    | Up to 5 developers              |
| Access to [Customer Community](https://github.com/openfaas/openfaas-pro)   | N/a      | Private access to [Customer Community](https://github.com/openfaas/openfaas-pro) for 2 named contacts per production cluster | As per Pro, for more named contacts |

Enterprise Support comes with an SLA, defined separately. Support for OpenFaaS Pro is on a self-service basis, with no formal SLA offered.

The [Customer Community](https://github.com/openfaas/openfaas-pro) is available to all customers and provides direct access to developers of OpenFaaS, and other customers for exclusive configurations & guides, early access to features, collaboration and announcements.

Did you know?

Any user of OpenFaaS is welcome to attend [a weekly Office Hours call](/community), which is a shared space for collaboration.

**Core features**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Scale to Zero | Not supported | [Global default, or custom time per function](/openfaas-pro/scale-to-zero) | As per Pro |
| Scale to From | Not supported | Supported, with additional checks for Istio | As per Pro |
| Build functions using private npm, Go and Pip modules | Not available | Enabled through build-time secrets | As Per Pro |
| Kubernetes service accounts for functions      | N/a             | [Supported per function](/reference/workloads) | As per Pro |
| Autoscaling strategy   | RPS-only | [CPU utilization, Capacity (inflight requests) or RPS](/architecture/autoscaling)      | As per Pro |
| Autoscaling granularity   | Single rule for all functions | Global defaults with override available per function      | As per Pro |
| UI Dashboard         | Legacy UI (in code-freeze)  | [New dashboard](/openfaas-pro/dashboard) with metrics, logs, CI integration and support for multiple namespaces | As per Pro |
| Async / queueing | In-memory only, max 10 items in queue, 256KB message size | NATS JetStream (new) | As per Pro |
| Metrics         | HTTP invocation metrics  | Plus CPU/RAM usage metrics and async/queue metrics      | As per Pro |
| CPU & RAM utilization | Not available | Integrated with Prometheus metrics, OpenFaaS REST API & CLI | As per Pro |
| Grafana Dashboards      | N/a             | 4x dashboards supplied in [Customer Community](https://github.com/openfaas/openfaas-pro) - overview, spotlight for debugging a function, queue-worker and Function Builder API | As per Pro |
| License                 | [MIT](https://github.com/openfaas/faas/blob/master/LICENSE)             | [Commercial license EULA](https://github.com/openfaas/faas/blob/master/pro/EULA)     | As per Pro |

> Did you know? Synadia, the vendor of NATS Streaming announced the product is now deprecated, and it will receive no updates from June 2023 onwards. OpenFaaS Ltd developed an alternative based upon their newest product JetStream. [Learn more about JetStream for OpenFaaS](https://docs.openfaas.com/openfaas-pro/jetstream/)

**Event-connectors**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| [Kafka event trigger](/openfaas-pro/kafka-events) | Not supported | Supports SASL or TLS auth, Aiven, Confluent and self-hosted | As per Pro |
| [Postgres trigger](/openfaas-pro/postgres-events) | Not supported | Supports insert, update and delete, with table-level filters using WAL or LISTEN/NOTIFY. | As per Pro |
| [AWS SQS trigger](/openfaas-pro/sqs-events) | Not supported | Supported | As per Pro |
| [Cron and scheduled invocations](/reference/cron) | Community support | Community support | Full support |

**Durability and reliability**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Health checks | Not supported | [Health checks](/reference/workloads) supported with custom HTTP path and intervals per function | As per Pro |
| Retries for failed function invocations | Not supported | [Retry invocations](/openfaas-pro/retries) for configured HTTP codes with an exponential back-off | As per Pro |
| Highly Available messaging | Not available, in-memory only | Available for NATS JetStream, with 3x servers. | As per Pro |
| Long executions of async functions | Limited to 5 minutes | Configurable duration | As per Pro |
| Callback to custom URL for async functions | Supported | As per CE | As per CE |

**Security**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Authentication for OpenFaaS API, CLI and UI | Shared admin password between everyone who uses OpenFaaS | [Sign-On with OIDC Okta/Auth0](/openfaas-pro/sso) | [Custom Single Sign-On with your own OIDC-compatible IdP](/openfaas-pro/sso) |
| Compatibility with Istio for mTLS | N/a | Supported | As per Pro |
| PCI/GDPR compliance       | Sensitive information such as the request body/response body, headers may be printed into the logs for each asynchronous invocation | Sensitive information is not printed to the logs for asynchronous requests | As per Pro |
| Secure isolation with Kata containers or gVisor      | N/a             | Supported using an [OpenFaaS Pro Profile and runtimeClass](/reference/profiles) | As per Pro |
| [Service links](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#accessing-the-service) injected as environment variables | Yes, cannot be disabled | Disabled as a default | As per Pro |
| [Pod privilege escalation](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) | Default for Kubernetes | Explicitly disabled | As per Pro |
| Split installation without ClusterAdmin role | N/a | Provided in [Customer Community](https://github.com/openfaas/openfaas-pro) | As per Pro | 

**Platform features**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Deploy functions via REST API | Yes | As per CE | As per CE | 
| Build containers and functions via REST API | N/a | [Yes via Function Builder API](/openfaas-pro/builder) | As per Pro |
| Multiple namespace support | No support | Supported with Kubernetes namespaces | As per Pro |

### Trusted by

OpenFaaS Pro is trusted by:

* Fortune 500 company (semiconductor / global technology)
* Fortune 500 company (financial services)
* Check Point Software Technologies Ltd (Global 2000 company)
* Yokogawa Electric Corporation (Global 2000 company)
* Waylay.io
* RateHub.ca
* Surge "workwithsurge"
* Edge Delta
* Patchworks Integration Limited
* Klar MX

Tell us about your use-case for OpenFaaS Pro, CE or faasd and see what other companies are doing in the: [ADOPTERS file](https://github.com/openfaas/faas/blob/master/ADOPTERS) 

### Support

OpenFaaS Enterprise comes with support for the Certified Open Source Components and OpenFaaS Pro, with support via email within a Service Level Agreement (SLA). Along with email support, each team gets their own dedicated Slack channel for questions, collaboration and assistance.

OpenFaaS Pro operates on a self-service model with support for OpenFaaS Pro features only via email.

Both tiers come with access to the [Customer Community](https://github.com/openfaas/openfaas-pro), for feedback & collaboration with the OpenFaaS developers and other customers. 

No support is offered to commercial users of OpenFaaS CE, which is primarily meant for exploration and open source developers.

### Getting started

OpenFaaS Pro is developed primarily for team deploying to Kubernetes, however most of the features are also compatible with faasd. Learn about faasd and OpenFaaS Pro here: [The Event-Driven Edge with OpenFaaS](https://www.openfaas.com/blog/eventdriven-edge/)

Are you interested in OpenFaaS for your organisation? [Contact us](https://openfaas.com/support/) to find out more.
