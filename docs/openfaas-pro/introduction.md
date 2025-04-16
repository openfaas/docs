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

### Comparison

Are you looking for a comparison of features between OpenFaaS CE, OpenFaaS Edge, OpenFaaS Standard and OpenFaaS for Enterprises? [See the comparison table](/openfaas-pro/comparison/).

### Support

A separate Enterprise Support Agreement is required for OpenFaaS for Enterprises, which covers the Certified Open Source Components and OpenFaaS Pro features. Support is provided via email within a Service Level Agreement (SLA). Along with email support, each team gets their own dedicated Slack channel for questions, collaboration and assistance.

OpenFaaS Standard is an off-the-shelf product that operates on a self-service model with support for OpenFaaS Pro features only via email.

Both tiers come with access to the [Customer Community](https://github.com/openfaas/customers), for feedback & collaboration with the OpenFaaS developers and other customers. 

No support is offered for OpenFaaS CE, which is primarily meant for personal use and for exploration prior to a purchase.

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
* Built-in function authentication with IAM for OpenFaaS (Q2 2024)
* Custom Autoscaling strategies for RAM, and any other Prometheus metrics (Q3 2024)
* Go SDK for Function Builder API (Q3 2024)
* OpenFaaS Edge - commercial faasd distribution with support, and reselling agreement (Q4 2024)
* RabbitMQ Connector (Q4 2024)
* Go SDK for the Function Builder API (Q3 2024)
* Asynchronous function cancellation through X-Call-ID identifier (Q4 2024)
* Reference architecture for spot instances on AWS EKS with Karpenter (Q1 2025)
* Websocket support for OpenFaaS Standard (Q1 2025)
* GPU support for OpenFaaS Edge (Q2 2025)

Upcoming:
* Event connector for - Azure Service Bus and Google PubSub (added upon request)
* Conversion to structured/JSON logging of OpenFaaS Pro components (ongoing - 80% complete)
* Switching to queue per function, including queued-based scaling (R&D phase)

Is there something else you need for your team or organisation? [Get in touch with us here](https://openfaas.com/support/).

If you're already a customer, we welcome suggestions for the roadmap in [the Customer Community](https://github.com/openfaas/customers).

### Getting started

OpenFaaS Standard is developed primarily for teams deploying to Kubernetes, however most of the features are also compatible with OpenFaaS Edge.

Learn about OpenFaaS Edge and OpenFaaS Standard here: [Introducing OpenFaaS Edge](https://www.openfaas.com/blog/meet-openfaas-edge/)

Are you interested in OpenFaaS for your organisation? [Contact us](https://openfaas.com/pricing/) to contact us or to find out more.

