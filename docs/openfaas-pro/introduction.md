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
* GitOps compatibility with the Function CustomResource, with Helm, ArgoCD and FluxCD.

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

### Grafana dashboards

OpenFaaS comes with [built-in Prometheus metrics](/architecture/metrics). We provide [a collection of Grafana dashboards](/openfaas-pro/grafana-dashboards) to customers to help with monitoring OpenFaaS and individual functions.


### Private code from functions

Get access to Pro plugins for the faas-cli and additional function templates to build functions that require code from private Git, PyPy or NPM.

* Use private npm, python, go modules from functions with [faas-cli plugins and build-time secrets](/cli/build/#plugins-and-build-time-secrets)

### On our roadmap

Recently released:

* Scaling upon inflight requests for long running & memory/CPU bound functions (released Jan 2022)
* A new Pro UI dashboard for managing and monitoring OpenFaaS functions across namespaces (released March 2022)
* CPU and RAM usage metrics within the OpenFaaS API, CLI and Pro UI dashboard (released Feb 2022)
* AWS SQS event-connector (released January 2022)
* Concurrency limiting for functions - i.e. one request per container
* Redesigned async system with NATS JetStream which replaces NATS Streaming (released)
    * NATS Streaming is available for CE and will be deprecated in June 2023.
    * Dedicated Helm chart for installing additional queue-workers with JetStream
    * Structured logging / JSON for OpenFaaS Pro customers in the queue-worker
* Custom readiness per function for enhanced concurrency limiting (released Oct 2022)
* Postgres event connector using either WAL or LISTEN/NOTIFY (released Dec 2022)
* Access to private artifact registries from `faas-cli build` for npm, Pip, Go modules, etc (released Dec 2022)
* AWS SNS event-connector (released Jan 2023)

Upcoming:

* Ongoing conversion to structured/JSON logging of OpenFaaS Pro components
* IAM & RBAC for the OpenFaaS REST API
* Webhook-based auditing of the OpenFaaS REST API
* Additional event triggers i.e. AMQP event trigger for RabbitMQ and Azure Service Bus
* Proactive remote monitoring for support customers
* Enhanced multi-tenant isolation for large organisations and service providers

Is there something else you need for your team or organisation? [Get in touch with us here](https://openfaas.com/support/).

If you're already a customer, we welcome suggestions for the roadmap in [the Customer Community](https://github.com/openfaas/customers).

### Comparison

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers and initial exploration of functions, OpenFaaS Pro is meant for production.

OpenFaaS Pro is a distribution of OpenFaaS with additional features and configurations that we believe customers need to operate a product or service in production.

As we tune OpenFaaS Pro for our existing customers, we improve it for everyone else at the same time - writing articles, tuning configuration and adding key features.

We see it as the start of a two-way relationship and an opportunity to collaborate directly with our team.

**Support**

| Description                 | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS for Enterprise             |
| ----------------------------| ------------------|------------------------|---------------------------------|
| Suitability                 | Open Source developers and initial exploration  | Production, business critical, or PoC | Regulated companies which may have additional legal and compliance requirements |
| SLA                         | N/a               | N/a                    | Response within 1 business day for P1 |
| Buying process              | N/a           | Invoice paid by bank transfer           | Supplier portals, custom paperwork, negotiation with procurement. |
| Legal review of contract    | N/a           | N/a                     | Yes |
| Signing of Mutual NDA       | N/a               | N/a                    | Subject to agreement |
| Additional compliance needs | N/a         | N/a                    | Subject to agreement |
| Support via email           | N/a               | Pro features only      | All certified Open Source and commercial components |
| Support via GitHub          | N/a               | Pro features only using the Customer Community      | N/a |
| Support via Slack           | N/a               | N/a                    | Up to 5 developers              |
| License                     | [MIT](https://github.com/openfaas/faas/blob/master/LICENSE)             | [Commercial license EULA](https://github.com/openfaas/faas/blob/master/pro/EULA)     | As per Pro |
| Architecture review  | N/a            | N/a                    | With our team via Zoom |
| Onboarding call | N/a            | N/a                    | With our team via Zoom |
| [Customer Community](https://github.com/openfaas/customers)   | N/a      | Private access to [Customer Community](https://github.com/openfaas/customers) - one user per licensed cluster | Custom amount of users |

OpenFaaS For Enterprise comes with an SLA, defined separately. It is suitable for companies which have a separate legal and procurement department, who are regulated and have additional legal or compliance requirements. The annual architecture review is to reduce risk by reviewing the configuration and informing the team of any recommended changes to the installation and configuration of OpenFaaS.

Support for OpenFaaS Pro is on a self-service basis, with no formal SLA offered.

The [Customer Community](https://github.com/openfaas/customers) is a private GitHub repository for giving feedback to the OpenFaaS team, for early access to new features and collaboration with other customers.

Did you know?

The OpenFaaS community holds a weekly [Office Hours call](/community) on Zoom where you can start contributing to the project and ask any additional questions you may have.

**Autoscaling**

Did you know? OpenFaaS Pro's autoscaling engine can scale many different types of functions and closely match the load with the right amount of replicas.

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Scale to Zero | Not supported | [Global default, or custom time per function](/openfaas-pro/scale-to-zero) | As per Pro |
| Maximum replicas per function | 5 Pods | No limit applied | As per Pro |
| Scale to From | Not supported | Supported, with additional checks for Istio | As per Pro |
| Autoscaling strategy   | RPS-only | [CPU utilization, Capacity (inflight requests) or RPS](/architecture/autoscaling)      | As per Pro |
| Autoscaling granularity   | Single rule for all functions | Configurable per function | As per Pro |

Data-driven, intensive, or long running functions are best suited to capacity-based autoscaling, which is only available in OpenFaaS Pro.

Scaling to zero is also a Pro feature, which can be opted into on a per function basis, with a custom idle threshold.

**Core features**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| UI Dashboard         | Legacy UI (in code-freeze)  | [New UI dashboard](/openfaas-pro/dashboard) with metrics, logs & CI integration | As per Pro, but with support for multiple namespaces |
| Consume secrets in `faas-cli build` for npm, Go and Pypy | Not available | Via build-time secrets | As Per Pro |
| Kubernetes service accounts for functions      | N/a             | [Supported per function](/reference/workloads) | As per Pro |
| Async / queueing | In-memory only, max 10 items in queue, 256KB message size | JetStream with shared queue | JetStream with dedicated queues |
| Metrics         | Basic function metrics  | Function, HTTP, CPU/RAM usage, and async/queue metrics      | As per Pro |
| CPU & RAM utilization | Not available | Integrated with Prometheus metrics, OpenFaaS REST API & CLI | As per Pro |
| Grafana Dashboards      | N/a             | 4x dashboards supplied in [Customer Community](https://github.com/openfaas/customers) - overview, spotlight for debugging a function, queue-worker and Function Builder API | As per Pro |
| GitOps & CRD support | N/a | ArgoCD & FluxCD compatibility using the Function CRD | As per Pro |
| Deployment options | faas-cli, or REST API | faas-cli, REST API, kubectl or GitOps solution using Function CRD | As per Pro |

> Did you know? Synadia, the vendor of NATS Streaming announced the product is now deprecated, and it will receive no updates from June 2023 onwards. OpenFaaS Ltd developed an alternative based upon their newest product JetStream. [Learn more about JetStream for OpenFaaS](https://docs.openfaas.com/openfaas-pro/jetstream/)

**Event-connectors**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Number of topics per function   | One topic per function | Multiple topics per function | As per Pro |
| [Kafka event trigger](/openfaas-pro/kafka-events) | Not supported | Supports SASL or TLS auth, Aiven, Confluent and self-hosted | Support with SLA |
| [Postgres trigger](/openfaas-pro/postgres-events) | Not supported | Supports insert, update and delete, with table-level filters using WAL or LISTEN/NOTIFY. | Support with SLA |
| [AWS SQS trigger](/openfaas-pro/sqs-events) | Not supported | Standard support | Support with SLA |
| [Cron and scheduled invocations](/reference/cron) | Community support | Standard support | Support with SLA |

**Durability and reliability**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Readiness probes | Not supported | [Readiness probes](/reference/workloads) supported with custom HTTP path and intervals per function | As per Pro |
| Retries for failed function invocations | Not supported | [Retry invocations](/openfaas-pro/retries) for configured HTTP codes with an exponential back-off | As per Pro |
| Highly Available messaging | Not available, in-memory only | Available for NATS JetStream, with 3x servers. | As per Pro |
| Long executions of async functions | Limited to 5 minutes | Configurable duration | As per Pro |
| Callback to custom URL for async functions | Supported | As per CE | As per CE |

**Security**

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Authentication for OpenFaaS API, CLI and UI | Shared admin password between everyone who uses OpenFaaS | N/a | [Single Sign-On with OIDC](/openfaas-pro/sso) |
| Compatibility with Istio for mTLS | N/a | Supported | As per Pro |
| PCI/GDPR compliance       | Sensitive information such as the request body/response body, headers may be printed into the logs for each asynchronous invocation | Sensitive information is not printed to the logs for asynchronous requests | As per Pro |
| Secure isolation with Kata containers or gVisor      | N/a             | N/a | Supported using an [OpenFaaS Pro Profile and runtimeClass](/reference/profiles) |
| [Service links](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#accessing-the-service) injected as environment variables | Yes, cannot be disabled | Disabled as a default | As per Pro |
| [Pod privilege escalation](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) | Default for Kubernetes | Explicitly disabled | As per Pro |
| Split installation without ClusterAdmin role | N/a | Provided in [Customer Community](https://github.com/openfaas/customers) | As per Pro | 

Isolation using Kata containers or gVisor is advisable when running untrusted code, or when you want to ensure that your functions are not vulnerable to container escape attacks.

**Platform features**

We have several customers of varying size who host code on behalf of their customers. OpenFaaS can provide a complete workflow from building the code securely, to deploying it within an isolated namespace. It's easy to integrate with existing systems using the REST APIs we make available.

| Description           | OpenFaaS CE       | OpenFaaS Pro           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Deploy functions via REST API | Yes | As per CE | As per CE | 
| Build containers and functions via REST API | N/a | N/a | [Yes via Function Builder API](/openfaas-pro/builder) |
| Multiple namespace support | No support | N/a | Supported with Kubernetes namespaces |

Some customers extend their own platform using OpenFaaS functions, because it's quicker and easier to deploy a function than change the core platform.

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
* Live Time Value (LTV) Co.
* BCubed Engineering
* Kubiya

Tell us about your use-case for OpenFaaS Pro, CE or faasd and see what other companies are doing in the: [ADOPTERS file](https://github.com/openfaas/faas/blob/master/ADOPTERS) 

### Support

OpenFaaS for Enterprise comes with support for the Certified Open Source Components and OpenFaaS Pro, with support via email within a Service Level Agreement (SLA). Along with email support, each team gets their own dedicated Slack channel for questions, collaboration and assistance.

OpenFaaS Pro operates on a self-service model with support for OpenFaaS Pro features only via email.

Both tiers come with access to the [Customer Community](https://github.com/openfaas/customers), for feedback & collaboration with the OpenFaaS developers and other customers. 

No support is offered to commercial users of OpenFaaS CE, which is primarily meant for exploration and open source developers.

### Getting started

OpenFaaS Pro is developed primarily for team deploying to Kubernetes, however most of the features are also compatible with faasd. Learn about faasd and OpenFaaS Pro here: [The Event-Driven Edge with OpenFaaS](https://www.openfaas.com/blog/eventdriven-edge/)

Are you interested in OpenFaaS for your organisation? [Contact us](https://openfaas.com/support/) to find out more.
