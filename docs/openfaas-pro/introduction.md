## OpenFaaS Pro

OpenFaaS Pro is a commercially licensed distribution of OpenFaaS with additional features, configurations and commercial support from the founders.

There are two editions of OpenFaaS Pro - Standard and for Enterprises.

Standard is meant for any team making use of OpenFaaS at work, whether external, or internal-facing. OpenFaaS for Enterprises is meant for multi-tenancy, and regulated companies who have additional requirements around security, compliance and support.

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is meant for open-source developers and basic exploration, it is not licensed for commercial usage beyond a one-off 60-day trial. OpenFaaS Pro is battle-tested and run in production by companies of various sizes.

### Core platform

Efficiency and redundancy:

* [New auto-scaling engine](/architecture/autoscaling/) to get the scaling just right either on Requests Per Second (RPS), Inflight requests or CPU.
* [Scale idle functions to zero](/openfaas-pro/scale-to-zero) to save on compute costs, increase efficiency and lower your threat profile
* [Retry failed invocations for functions](/openfaas-pro/retries) to handle issues with downstream APIs and back-pressure on concurrency-limited functions
* CPU and RAM usage metrics for every function to measure usage and fine-tune limits
* [GitOps compatibility with the Function CustomResource, with Helm, ArgoCD and FluxCD](/openfaas-pro/function-crd)

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
* [Trigger functions from AWS SQS](/openfaas-pro/sqs-events) to integrate with events from AWS services via a managed queue.
* [Trigger functions from AWS SNS](/openfaas-pro/sns-events) to integrate with events from AWS managed services via pub/sub.

### Workload tuning

To avoid errors when scaling up or down, you may need to tune your function's configuration to suit how it works. Pro users get access to fine-tune health checks, set custom Kubernetes service accounts and termination grace periods for shutting down functions.

* Custom HTTP health checks for functions - including path, period seconds and initial delay
* Custom Kubernetes service accounts for functions to access the Kubernetes API
* Custom runtime profiles for security & isolation using gVisor, kata containers etc.
* Custom TerminationGracePeriod for draining work for long running functions
* Custom support for probing Istio endpoints during scale from zero

Read more: [OpenFaaS workloads](/reference/workloads/) and [OpenFaaS Profiles](/reference/profiles/#use-an-alternative-runtimeclass)

### Enterprise security

* [Airgapped/offline installations and image mirroring](/openfaas-pro/airgap)
* [Identity and Access Management (IAM)](/openfaas-pro/iam/overview)
* [Single Sign On with OpenID Connect (OIDC)](/openfaas-pro/sso/overview)

[Learn more about IAM for OpenFaaS](/openfaas-pro/iam/overview)

### Platform building features

Run a secure, multi-tenant functions platform - for internal or external users:

* [Build a Multi-Tenant Functions Platform with OpenFaaS](https://www.openfaas.com/blog/build-a-multi-tenant-functions-platform/)

Build functions at scale - for services providers and large teams:

* [Build functions via REST API](/openfaas-pro/builder) using source code without the need to create and manage dozens or hundreds of independent CI jobs.

Integrate securely using short-lived tokens from your Identity Provider (IdP) or from Kubernetes:

* [How to authenticate to the OpenFaaS API using Kubernetes JWT tokens](https://www.openfaas.com/blog/kubernetes-tokens-openfaas-api/)

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
* IAM & RBAC for the OpenFaaS REST API (Released June 2023)
* Enhanced multi-tenant isolation for large organisations and service providers (Released June 2023)
* Invocation via the OpenFaaS Pro Dashboard (Oct 2023)
* Webhook-based auditing of the OpenFaaS REST API (Dec 2023)
* Webhook-based billing metrics per invocation for billing your users  (Dec 2023)
* Stream Server Sent Events (SSE) from functions i.e. OpenAI (Jan 2024)
* Proactive remote monitoring for support customers (live in pilot)
* Official C# .NET 8 template (Apr 2024)
* `airfaas` CLI plugin for airgapped installations and mirroring images to private registries (Apr 2024)
* GPU scheduling support for functions via Profiles i.e. `nvidia.com/gpu: 1` (Mar 2024)

Upcoming:
* Built-in function authentication with IAM for OpenFaaS (in preview)
* Conversion to structured/JSON logging of OpenFaaS Pro components (ongoing - 80% complete)
* Dynamic, dedicated Async queues for each function, including queued-based scaling (R&D phase)
* Go SDK for the Function Builder API
* Additional event triggers i.e. AMQP event trigger for RabbitMQ and Azure Service Bus and Google PubSub (added upon request)

Is there something else you need for your team or organisation? [Get in touch with us here](https://openfaas.com/support/).

If you're already a customer, we welcome suggestions for the roadmap in [the Customer Community](https://github.com/openfaas/customers).

### Comparison

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is licensed for open-source developers and initial exploration of functions, OpenFaaS Pro is licensed for production.

OpenFaaS Pro is a distribution of OpenFaaS with additional features and configurations that we believe customers need to operate a product or service in production.

As we tune OpenFaaS Pro for our existing customers, we improve it for everyone else at the same time - writing articles, tuning configuration and adding key features.

**Support**

All the items below are included in the standard self-service/license-only package, unless specifically stated as requiring an Enterprise Service Agreement.

| Description                 | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------------| ------------------|------------------------|---------------------------------|
| Suitability                 | PoCs, non-production, experimentation & non-commercial use only  | Production, business critical, or PoC | Regulated companies which may have additional legal and compliance requirements |
| SLA                         | n/a               | n/a                    | Response within 1 business day for P1 (with ESA) |
| Buying process              | n/a           | Monthly via credit card, or via annual invoice (SWIFT/ACH)    | Annual invoice/purchase order (SWIFT/ACH). |
| Legal review of contract/red-lining    | n/a           | n/a                     | Subject to agreement |
| Signing of Mutual NDA       | n/a               | n/a                    | Subject to agreement |
| Additional compliance needs | n/a         | n/a                    | Subject to agreement |
| Support via email           | n/a               | Pro features only      | All certified Open Source and commercial components |
| Support via GitHub          | n/a               | Pro features only using the Customer Community      | n/a |
| Support via Slack           | n/a               | n/a                    | 5x users (with ESA) |
| License                     | [Community Edition EULA - 60 day limit for commercial use](https://github.com/openfaas/faas/blob/master/EULA.md)| [Commercial license EULA](https://github.com/openfaas/faas/blob/master/pro/EULA)     | as per Standard |
| Annual architecture review  | n/a            | n/a                    | With our team via Zoom (with ESA) |
| Onboarding call | n/a            | With our team via Zoom                   | As per Standard |
| Developer licenses | n/a            | n/a                   | 5x users (for local development) |
| [Customer Community](https://github.com/openfaas/customers)   | n/a      | One user per licensed cluster | Custom amount of users |

**Self-service / licensing only**

Both OpenFaaS Standard and OpenFaaS for Enterprises are available on a self-service basis without a defined Service Level Agreement (SLA). This tier includes licenses, basic email support and collaboration through the Customer Community.

**Enterprise Support Agreement (ESA)**

A separate Enterprise Support Agreement (ESA) can be purchased for OpenFaaS for Enterprises which includes an SLA, access to OpenFaaS Ltd engineering via Slack, and an annual architecture review. This package is suitable for companies which have a separate legal and procurement department, who are regulated and have additional legal or compliance requirements. An annual architecture review is included with this package to reduce risk by reviewing the configuration and informing the team of any recommended changes to the installation and configuration of OpenFaaS.

The [Customer Community](https://github.com/openfaas/customers) is a private GitHub repository for giving feedback to the OpenFaaS team, for early access to new features and collaboration with other customers.

Did you know?

The OpenFaaS community holds a weekly [Office Hours call](/community) on Zoom where you can start contributing to the project and ask any additional questions you may have.

**Autoscaling**

Did you know? OpenFaaS Pro's autoscaling engine can scale many different types of functions and closely match the load with the right amount of replicas.

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Scale to Zero | Not supported | [Global default, or custom time per function](/openfaas-pro/scale-to-zero) | as per Standard |
| Maximum replicas per function | 5 Pods | No limit applied | as per Standard |
| Scale from Zero | Not supported | Supported, with additional checks for Istio | as per Standard |
| Autoscaling strategy   | RPS-only | [CPU utilization, Capacity (inflight requests) or RPS](/architecture/autoscaling)      | as per Standard |
| Autoscaling granularity   | Single rule for all functions | Configurable per function | as per Standard |

Data-driven, intensive, or long running functions are best suited to capacity-based autoscaling, which is only available in OpenFaaS Pro.

Scaling to zero is also a commercial feature, which can be opted into on a per function basis, with a custom idle threshold.

**Core features**

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Maximum functions     | 15                | 500                    | 5000 per namespace              |
| Maximum namespaces    | 1                 | 1                      | Unlimited                       |
| Public/private registries   | Public images only, without authentication. | Public or private with authentication | As per Standard |
| UI Dashboard         | Legacy UI (in code-freeze)  | [New UI dashboard](/openfaas-pro/dashboard) with metrics, logs & CI integration | as per Standard, but with support for multiple namespaces |
| Consume secrets in `faas-cli build` for npm, Go and Pypy | Not available | Via build-time secrets | as per Standard |
| Kubernetes service accounts for functions      | n/a             | [Supported per function](/reference/workloads) | as per Standard |
| Async / queueing | In-memory only, max 10 items in queue, 256KB message size | JetStream with shared queue | JetStream with dedicated queues |
| Metrics         | Basic function metrics  | Function, HTTP, CPU/RAM usage, and async/queue metrics      | as per Standard |
| CPU & RAM utilization | Not available | Integrated with Prometheus metrics, OpenFaaS REST API & CLI | as per Standard |
| Grafana Dashboards      | n/a             | 4x dashboards supplied in [Customer Community](https://github.com/openfaas/customers) - overview, spotlight for debugging a function, queue-worker and Function Builder API | as per Standard |
| GitOps & CRD support | Not available | ArgoCD, FluxCD, Helm and Kustomize compatibility using the Function CRD | as per Standard |
| Deployment options | faas-cli or REST API | As per CE, plus: Function CRD with kubectl, Helm or GitOps | as per Standard |
| Custom Resource Definition | Not available | Function and Profile CRDs | as per Standard |
| Image Pull Policy (for air-gap) | Always | `Always`, `IfNotPresent` or `Never` | as per Standard |

> Did you know? Synadia, the vendor of NATS Streaming announced the product is now deprecated, and it will receive no updates from June 2023 onwards. OpenFaaS Ltd developed an alternative based upon their newest product JetStream. [Learn more about JetStream for OpenFaaS](https://docs.openfaas.com/openfaas-pro/jetstream/)

Learn how to deploy functions via kubectl: [Function CRD](/openfaas-pro/function-crd)

**Advanced function configuration**

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Toleration support    | n/a               | Supported via Profiles | As per Standard |
| Taints support        | n/a               | Supported via Profiles | As per Standard |
| Custom podSecurityContext    | n/a        | Supported via Profiles | As per Standard |
| Custom runtimeClassName    | n/a        | n/a | As per Standard |
| Custom topologySpreadConstraints    | n/a        | Supported via Profiles | As per Standard |
| Custom affinity/anti-affinity    | n/a        | Supported via Profiles | As per Standard |
| Custom deployment strategy | n/a | Supported via Profiles | As per Standard |
| Additional resources (e.g device plugin managed resources) | n/a | Supported via Profiles | As per standard |
| nodeSelector | n/a | Available via `constraints` in Function spec | As per Standard |
| Custom DNS configuration | n/a | Supported via Profiles | As per Standard |

**IAM and Policy**

Identity Access Management (IAM) and Policy-based authorization is available for OpenFaaS for Enterprises customers. It enables fine-grained policy, along with federation to external identity providers and Single Sign-On (SSO) with OpenID Connect (OIDC). 

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Authentication        | Basic auth.       | As per CE              | [SSO with OIDC](https://docs.openfaas.com/openfaas-pro/sso/overview/)           |
| Authorization         | n/a               | n/a                    | [Policy-based identity and access management](https://docs.openfaas.com/openfaas-pro/iam/overview/) |
| OIDC Federation       | n/a               | n/a                    | Federate external OIDC providers for authorization |
| Authorization with Kubernetes Service Account | n/a | n/a | [Supported via projected JWT tokens](https://www.openfaas.com/blog/kubernetes-tokens-openfaas-api/) |
| Keyless deployment from CI/CD | n/a | n/a | GitLab CI, GitHub Actions, and any OIDC compatible source |

**Event-connectors**

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Number of topics per function   | One topic per function | Multiple topics per function | as per Standard |
| [Kafka event trigger](/openfaas-pro/kafka-events) | Not supported | Supports SASL or TLS auth, Aiven, Confluent and self-hosted | Support with SLA |
| [Postgres trigger](/openfaas-pro/postgres-events) | Not supported | Supports insert, update and delete, with table-level filters using WAL or LISTEN/NOTIFY. | Support with SLA |
| [AWS SQS trigger](/openfaas-pro/sqs-events) | Not supported | Standard support | Support with SLA |
| [Cron and scheduled invocations](/reference/cron) | Community support | Standard support | Support with SLA |

**Durability and reliability**

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Readiness probes | Not supported | [Readiness probes](/reference/workloads) supported with custom HTTP path and intervals per function | as per Standard |
| Retries for failed function invocations | Not supported | [Retry invocations](/openfaas-pro/retries) for configured HTTP codes with an exponential back-off | as per Standard |
| Highly Available messaging | Not available, in-memory only | Available for NATS JetStream, with 3x servers. | as per Standard |
| Long executions of async functions | Limited to 5 minutes | Configurable duration | as per Standard |
| Callback to custom URL for async functions | Supported | As per CE | As per CE |

**Security**

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Air-gap and offline support | No, Internet connection required | Supported | As per Standard |
| [faas-cli airfaas](https://www.openfaas.com/blog/airgap-serverless-functions/)               | n/a              | Mirror images and install in offline environments | As per Standard | 
| Authentication for OpenFaaS API, CLI and UI | Shared admin password between everyone who uses OpenFaaS | as per CE | [Single Sign-On with OIDC](https://docs.openfaas.com/openfaas-pro/iam/overview/) |
| Identity and Policy | n/a | n/a | Least Privilege with [Identity and Access Management](https://docs.openfaas.com/openfaas-pro/iam/overview/) |
| Compatibility with Istio for mTLS | n/a | Supported | as per Standard |
| PCI/GDPR compliance       | Sensitive information such as the request body/response body, headers may be printed into the logs for each asynchronous invocation | Sensitive information is not printed to the logs for asynchronous requests | as per Standard |
| Secure isolation with Kata containers or gVisor      | n/a             | n/a | Supported using an [OpenFaaS Pro Profile and runtimeClass](/reference/profiles) |
| [Service links](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#accessing-the-service) injected as environment variables | Yes, cannot be disabled | Disabled as a default | as per Standard |
| [Pod privilege escalation](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) | Default for Kubernetes | Explicitly disabled | as per Standard |
| Split installation without ClusterAdmin role | n/a | Provided in [Customer Community](https://github.com/openfaas/customers) | as per Standard | 

Isolation using Kata containers or gVisor is advisable when running untrusted code, or when you want to ensure that your functions are not vulnerable to container escape attacks.

**Platform features**

We have several customers of varying size who host code on behalf of their customers. OpenFaaS can provide a complete workflow from building the code securely, to deploying it within an isolated namespace. It's easy to integrate with existing systems using the REST APIs we make available.

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Deploy functions via REST API | Yes | As per CE | As per CE | 
| Manage namespaces via REST API | n/a | n/a | Full CRUD API available |
| Build containers and functions via REST API | n/a | n/a | [Yes via Function Builder API](/openfaas-pro/builder) |
| Multiple namespace support | No support | n/a | Supported with Kubernetes namespaces |
| Multi-tenancy | Not supported | n/a | [Supported](https://www.openfaas.com/blog/build-a-multi-tenant-functions-platform/) |
| Billing metrics/chargeback | n/a | n/a | [Via webhooks](/openfaas-pro/billing-metrics/) |

Some customers extend their own platform using OpenFaaS functions, because it's quicker and easier to deploy a function than change the core platform.

### Support

OpenFaaS for Enterprises comes with support for the Certified Open Source Components and OpenFaaS Standard, with support via email within a Service Level Agreement (SLA). Along with email support, each team gets their own dedicated Slack channel for questions, collaboration and assistance.

OpenFaaS Standard operates on a self-service model with support for OpenFaaS Standard features only via email.

Both tiers come with access to the [Customer Community](https://github.com/openfaas/customers), for feedback & collaboration with the OpenFaaS developers and other customers. 

No support is offered to commercial users of OpenFaaS CE, which is primarily meant for exploration and open source developers.

### Getting started

OpenFaaS Standard is developed primarily for teams deploying to Kubernetes, however most of the features are also compatible with faasd. Learn about faasd and OpenFaaS Standard here: [The Event-Driven Edge with OpenFaaS](https://www.openfaas.com/blog/eventdriven-edge/)

Are you interested in OpenFaaS for your organisation? [Contact us](https://openfaas.com/support/) to find out more.
