# OpenFaaS stack

## Layers and responsibilities

The recommended platform for deploying OpenFaaS is [Kubernetes](https://kubernetes.io/), whether that is a local environment, a self-hosted cluster, or with a managed service such as [AWS Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/).

![Layers](https://github.com/openfaas/faas/raw/master/docs/of-layer-overview.png)

### CI / GitOps layer

OpenFaaS runs functions, but is also capable of running HTTP microservices. Each workload is built into a container image, and published in a container registry.

During development and evaluation, this is usually done manually using the `faas-cli`.

For production, there are a couple of popular options:

You can use the CI tooling built into your Source Control Management (SCM) system. GitHub Actions or a GitLab pipeline provide the easiest option to build and deploy functions by running `faas-cli deploy` or `faas-cli up` from within a job. Deployments are made after a job completes, changes are *pushed* into the cluster.

If you need to access a cluster within a private VPC or on-premises, this can be accomplished by [using a private and secure inlets tunnel](https://inlets.dev/blog/2021/12/06/private-deploy-from-github-actions.html).

As an alternative, a GitOps controller like [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) and [Flux](https://fluxcd.io/) can be used to continually update OpenFaaS or your OpenFaaS functions. A GitOps approach usually involves continuous deployment of new versions as soon as they are available. Deployments are made by pulling desired state from a special configuration repository.

### Application Layer

* [The OpenFaaS gateway](/architecture/gateway/) provides a REST API for managing functions, recording metrics and scaling them.
* [NATS](https://github.com/nats-io) is used for asynchronous function execution and queuing
* [Prometheus](https://prometheus.io/) provides metrics and enables the auto-scaling of the Community Edition and OpenFaaS Pro

With [OpenFaaS Pro](/openfaas-pro/introduction/), functions can be triggered via HTTP, Cron, AWS SQS or Apache Kafka.

The projects that make up OpenFaaS (Prometheus, Linux, OpenFaaS, NATS and Kubernetes) can be referred to as [The PLONK Stack](https://www.openfaas.com/blog/plonk-stack/). The PLONK stack is capable of running event-driven functions and traditional HTTP-based microservices.

These applications can be installed via Helm charts, or by using a GitOps operator like [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) or [Flux](https://fluxcd.io/).

### Infrastructure Layer

* The unit of execution for a function is a Pod, managed by containerd or Docker.
* A container registry holds each function as an immutable artifact that can be deployed to the OpenFaaS gateway using its REST API, UI or CLI.
* Kubernetes is the platform that allows functions to scale across multiple hosts, [faasd](/deployment/edge/) is a simpler alternative for smaller installations.

This layer is usually built manually during exploration and development, and using a tool like Terraform for production.

### Conceptual workflow

![Workflow](https://github.com/openfaas/faas/blob/master/docs/of-workflow.png?raw=true)

The Gateway can be accessed through its REST API, via the CLI or through the UI. All services or functions get a default route exposed, but custom domains can also be used for each endpoint.

Prometheus collects metrics which are available via the Gateway's API and which are used for auto-scaling.

By changing the URL for a function from `/function/NAME` to `/async-function/NAME` an invocation can be run in a queue using NATS Streaming. You can also pass an optional callback URL.

[faas-netes](https://github.com/openfaas/faas-netes/) is the most popular orchestration provider for OpenFaaS, but Docker Swarm, Hashicorp Nomad, AWS Fargate/ECS, and AWS Lambda have also been developed by the community. Providers are built with the [faas-provider SDK](https://github.com/openfaas/faas-provider).

See also: [Gateway architecture](/architecture/gateway/), [Autoscaling](/architecture/autoscaling/), [Asynchronous invocations](/reference/async/), and [Going to production](/architecture/production/)
