## Comparison of OpenFaaS editions

!!! info "Do we need the Community Edition or Pro?"

    OpenFaaS Community Edition (CE) is licensed for open-source developers and initial exploration of functions, OpenFaaS Pro is licensed for production.

OpenFaaS Pro is a distribution of OpenFaaS with additional features and configurations that we believe customers need to operate a product or service in production.

As we tune OpenFaaS Pro for our existing customers, we improve it for everyone else at the same time - writing articles, tuning configuration and adding key features.

**Support**

All the items below are included in the standard self-service/license-only package, unless specifically stated as requiring an Enterprise Service Agreement.

| Description                 | OpenFaaS CE       | OpenFaaS Edge | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------------| ------------------|---------------|------------------------|---------------------------------|
| Suitability                 | PoCs, non-production, experimentation & non-commercial use only | Event processing the edge, with low scaling requirements  | Production, business critical, or PoC | Regulated companies which may have additional legal and compliance requirements |
| Resale                      | No                | Subject to agreement | No                     | Subject to agreement |
| SLA                         | n/a               | n/a                  | n/a                    | Response within 1 business day for P1 (with ESA) |
| Buying process              | n/a               | Annual via invoice (SWIFT/ACH) | Monthly via credit card, or via annual invoice (SWIFT/ACH)    | Annual invoice/purchase order (SWIFT/ACH). |
| Legal review of contract/red-lining | n/a       | n/a                       | n/a                     | Subject to Enterprise Support Agreement |
| Signing of Mutual NDA       | n/a               | n/a                       | n/a                    | Subject to agreement |
| Additional compliance needs | n/a               |                       n/a | n/a                    | Subject to agreement |
| Support via email           | n/a               | Pro features only         | Pro features only      | All certified Open Source and commercial components |
| Support via private GitHub Discussions  | n/a   | Pro features only         | Pro features only      | n/a |
| Support via Slack           | n/a               | n/a                       | n/a                    | Requires Enterprise Support Agreement |
| License                     | [Community Edition EULA - 60 day limit for commercial use](https://github.com/openfaas/faas/blob/master/EULA.md) | [Commercial license EULA](https://github.com/openfaas/faas/blob/master/pro/EULA.md)  | [Commercial license EULA](https://github.com/openfaas/faas/blob/master/pro/EULA.md)     | as per Standard |
| Annual architecture review  | n/a            | n/a                            | n/a                    | With our team via Zoom (with ESA) |
| Onboarding call             | n/a            | With our team via Zoom | With our team via Zoom                   | As per Standard |
| Developer licenses          | n/a            | n/a                            | n/a                   | 5x users (for local development only) |
| [Customer Community](https://github.com/openfaas/customers)  | n/a      | One user per licensed host | One user per licensed cluster | Custom amount of users |

**Self-service / licensing only**

OpenFaaS Standard is available on a self-service basis without a defined Service Level Agreement (SLA). This tier includes licenses, basic email support and collaboration through the Customer Community.

The [Customer Community](https://github.com/openfaas/customers) is a private GitHub repository for giving feedback to the OpenFaaS team, for early access to new features and collaboration with other customers.

**Enterprise Support Agreement (ESA)**

A separate Enterprise Support Agreement (ESA) can be purchased for OpenFaaS for Enterprises which includes an SLA, access to OpenFaaS Ltd engineering via Slack, and an annual architecture review. This package is suitable for companies which have a separate legal and procurement department, who are regulated and have additional legal or compliance requirements. An annual architecture review is included with this package to reduce risk by reviewing the configuration and informing the team of any recommended changes to the installation and configuration of OpenFaaS.

Did you know?

The OpenFaaS community holds a weekly [Office Hours call](/community) on Zoom where you can start contributing to the project and ask any additional questions you may have.

**Autoscaling**

Did you know? OpenFaaS Pro's autoscaling engine can scale many different types of functions and closely match the load with the right amount of replicas.

| Description                   | OpenFaaS CE       | OpenFaaS Edge             | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ------------------------------| ------------------|---------------------------|------------------------|---------------------------------|
| Scale to Zero                 | Not supported     | Global timeout            | [Global timeout and override per function](/openfaas-pro/scale-to-zero) | as per Standard |
| Maximum replicas per function | 5                 | 1                         | No limit applied | as per Standard |
| Scale from Zero               | Not available     | Supported                 | Supported, with additional checks for Istio | as per Standard |
| Zero downtime updates         | Not available     | Not available             | Supported with readiness probes and rolling updates | as per Standard |
| Autoscaling strategy          | RPS               | Not applicable            | [CPU utilization, Capacity (inflight requests), RPS, async queue-depth and Custom (e.g. Memory)](/architecture/autoscaling)      | as per Standard |
| Autoscaling granularity       | One global rule   | Not applicable            | Configurable per function | as per Standard |

Data-driven, intensive, or long running functions are best suited to capacity-based or queue-based autoscaling, which is only available in OpenFaaS Pro.

Scaling to zero is also a commercial feature, which can be opted into on a per function basis, with a custom idle threshold.

**Core features**

| Description           | OpenFaaS CE       | OpenFaaS Edge | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|--------------|------------------------|---------------------------------|
| Maximum functions     | 15                | 25-250 (per host) | 500                    | 5000 per namespace               |
| Maximum namespaces    | 1                 | Unlimited         | 1                      | Unlimited                                   |
| High availability for control-plane     | n/a               | n/a               | Leader election for Kubernetes operator | as Per Standard |
| High availability for functions | Multiple replicas supported | n/a | as per Standard | as per Standard | as per Standard |
| Public/private registries   | Unauthenticated public repositories only. | Public or private with auth. |Public or private with auth. | As per Standard |
| UI Dashboard         | Legacy UI (in code-freeze)  | Dashboard is an optional add-on | [New UI dashboard](/openfaas-pro/dashboard) with metrics, logs & CI integration | as per Standard, but with support for multiple namespaces |
| Consume secrets in `faas-cli build` for npm, Go and Pypy | Not available | Via build-time secrets | Via build-time secrets | as per Standard |
| Kubernetes service accounts for functions      | n/a             | n/a | [Supported per function](/reference/workloads) | as per Standard |
| Async / queueing | In-memory only, max 10 items in queue, 256KB message size | JetStream (shared queue) | JetStream (shared queue)  | JetStream (dedicated queues) |
| Metrics         | Basic function metrics  | As per Standard  | Function, HTTP, CPU/RAM usage, and async/queue metrics      | as per Standard |
| CPU & RAM utilization | Not available  | As per Standard | Integrated with Prometheus metrics, OpenFaaS REST API & CLI | as per Standard |
| Grafana Dashboards      | n/a             | As per Standard | 4x dashboards supplied in [Customer Community](https://github.com/openfaas/customers) - overview, spotlight for debugging a function, queue-worker and Function Builder API | as per Standard |
| GitOps & CRD support | Not available | Not applicable | ArgoCD, FluxCD, Helm and Kustomize compatibility using the Function CRD | as per Standard |
| Deployment options | faas-cli or REST API | As per CE | As per CE, plus: Function CRD with kubectl, Helm or GitOps | as per Standard |
| Custom Resource Definition | Not available | Not applicable | Function and Profile CRDs | as per Standard |
| Image Pull Policy (for air-gap) | Always | As per Standard | `Always`, `IfNotPresent` or `Never` | as per Standard |
| GPU support | Not available | Available for core services | Available for functions via Profiles | as per Standard |

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

OpenFaaS Edge supports custom DNS servers.

**IAM and Policy**

Identity Access Management (IAM) and Policy-based authorization is available for OpenFaaS for Enterprises customers. It enables fine-grained policy, along with federation to external identity providers and Single Sign-On (SSO) with OpenID Connect (OIDC). 

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Authentication        | Basic auth.       | As per CE              | [SSO with OIDC](https://docs.openfaas.com/openfaas-pro/sso/overview/)           |
| Authorization         | n/a               | n/a                    | [Policy-based identity and access management](https://docs.openfaas.com/openfaas-pro/iam/overview/) |
| OIDC Federation       | n/a               | n/a                    | Federate external OIDC providers for authorization |
| Authorization with Kubernetes Service Account | n/a | n/a | [Supported via projected JWT tokens](https://www.openfaas.com/blog/kubernetes-tokens-openfaas-api/) |
| Keyless deployment from CI/CD | n/a | n/a | GitLab CI, GitHub Actions, and any OIDC compatible source |

OpenFaaS Edge supports Basic authentication only.

**Event-connectors**

| Description           | OpenFaaS CE       | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|------------------------|---------------------------------|
| Number of topics per function   | One topic per function | Multiple topics per function | as per Standard |
| [Kafka event trigger](/openfaas-pro/kafka-events) | Not supported | Supports SASL or TLS auth, Aiven, Confluent and self-hosted | Support with SLA |
| [Postgres trigger](/openfaas-pro/postgres-events) | Not supported | Supports insert, update and delete, with table-level filters using WAL or LISTEN/NOTIFY. | Support with SLA |
| [AWS SQS trigger](/openfaas-pro/sqs-events) | Not supported | Standard support | Support with SLA |
| [AWS SNS trigger](/openfaas-pro/sns-events) | Not supported | Standard support | Support with SLA |
| [RabbitMQ trigger](/openfaas-pro/rabbitmq-events) | Not supported | Standard support | Support with SLA |
| [Google Cloud Pub/Sub trigger](/openfaas-pro/pubsub-events) | Not supported | Standard support | Support with SLA |
| [Cron and scheduled invocations](/reference/cron) | Community support | Standard support, with structured logs | Support with SLA |

Event connectors can be purchased as a separate add-on for OpenFaaS Edge.

**Durability and reliability**

| Description           | OpenFaaS CE       | OpenFaaS Edge    | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|-------------------|------------------------|---------------------------------|
| Readiness probes | Not supported | Supported for scale from zero | [Readiness probes](/reference/workloads) supported with custom HTTP path and intervals per function | as per Standard |
| Retries for failed function invocations | Not supported | As per Standard | [Retry invocations](/openfaas-pro/retries) for configured HTTP codes with an exponential back-off | as per Standard |
| Highly Available messaging | No, in-memory only  | as per Standard | Durable configuration for NATS | as per Standard |
| Long executions of async functions | Limited to 5 minutes | As per Standard | Configurable duration | as per Standard |
| Callback to custom URL for async functions | Supported | As per CE | As per CE | As per CE |

**Security**

| Description           | OpenFaaS CE       | OpenFaaS Edge    | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|-------------------|------------------------|---------------------------------|
| Air-gap and offline support | No, Internet connection required | Supported | Supported | As per Standard |
| [faas-cli airfaas](https://www.openfaas.com/blog/airgap-serverless-functions/)               | n/a              | Mirror images and install in offline environments | As per Standard | As per Standard | 
| Authentication for OpenFaaS API, CLI and UI | Shared admin password between everyone who uses OpenFaaS | as per CE | As per CE | [Single Sign-On with OIDC](https://docs.openfaas.com/openfaas-pro/iam/overview/) |
| Identity and Policy | n/a | n/a | Least Privilege with [Identity and Access Management](https://docs.openfaas.com/openfaas-pro/iam/overview/) |
| Compatibility with Istio for mTLS | n/a | n/a | Supported | as per Standard |
| PCI/GDPR compliance       | Sensitive information such as the request body/response body, headers may be printed into the logs for each asynchronous invocation | As per Standard | Sensitive information is not printed to the logs for asynchronous requests | as per Standard |
| Secure isolation with Kata containers or gVisor      | n/a             | n/a | n/a | Supported using an [OpenFaaS Pro Profile and runtimeClass](/reference/profiles) |
| [Service links](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#accessing-the-service) injected as environment variables | n/a | Yes, cannot be disabled | Disabled as a default | as per Standard |
| [Pod privilege escalation](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) | Default for Kubernetes | n/a | Explicitly disabled | as per Standard |
| Split installation without ClusterAdmin role | n/a | n/a | Provided in [Customer Community](https://github.com/openfaas/customers) | as per Standard | 

Isolation using Kata containers or gVisor is advisable when running untrusted code, or when you want to ensure that your functions are not vulnerable to container escape attacks.

**Platform features**

We have several customers of varying size who host code on behalf of their customers. OpenFaaS can provide a complete workflow from building the code securely, to deploying it within an isolated namespace. It's easy to integrate with existing systems using the REST APIs we make available.

| Description           | OpenFaaS CE       | OpenFaaS Edge    | OpenFaaS Standard           | OpenFaaS for Enterprise             |
| ----------------------| ------------------|-------------------|------------------------|---------------------------------|
| Deploy functions via REST API | Yes | As per CE | As per CE | As per CE | As per CE | 
| Manage namespaces via REST API | n/a | REST API | n/a | REST API |
| Build containers and functions via REST API | n/a | By separate purchase of Function Builder API |  n/a | [Yes via Function Builder API](/openfaas-pro/builder) |
| Multiple namespace support | No support | containerd namespaces | n/a | Supported with Kubernetes namespaces |
| Multi-tenancy | Not supported | Available via iptables restrictions | n/a | [Supported](https://www.openfaas.com/blog/build-a-multi-tenant-functions-platform/) |
| Billing metrics/chargeback | n/a | n/a | n/a | [Via webhooks](/openfaas-pro/billing-metrics/) |

Some customers extend their own platform using OpenFaaS functions, because it's quicker and easier to deploy a function than change the core platform.
