## OpenFaaS stack

### Layers and responsibilities

The recommended platform for deploying OpenFaaS is [Kubernetes](https://kubernetes.io/), whether that is a local environment, a self-hosted cluster, or with a managed service such as [AWS Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/).

![Layers](https://github.com/openfaas/faas/raw/master/docs/of-layer-overview.png)

* [OpenFaaS Cloud](/openfaas-cloud/intro/) builds upon OpenFaaS to deliver GitOps with GitHub.com or GitLab self-hosted
* [NATS](https://github.com/nats-io) provides asynchronous execution and queuing
* [Prometheus](https://prometheus.io/) provides metrics and enables auto-scaling through [AlertManager](https://prometheus.io/docs/alerting/overview/)
* A container registry holds each immutable artifact that can be deployed on OpenFaaS via the API

The projects that make up OpenFaaS can be referred to as [The PLONK Stack](https://www.openfaas.com/blog/plonk-stack/).

### Conceptual workflow

![Workflow](https://github.com/openfaas/faas/blob/master/docs/of-workflow.png?raw=true)

The Gateway can be accessed through its REST API, via the CLI or through the UI. All services or functions get a default route exposed, but custom domains can also be used for each endpoint.

Prometheus collects metrics which are available via the Gateway's API and which are used for auto-scaling.

By changing the URL for a function from `/function/NAME` to `/async-function/NAME` an invocation can be run in a queue using NATS Streaming. You can also pass an optional callback URL.

[faas-netes](https://github.com/openfaas/faas-netes/) is the most popular orchestration provider for OpenFaaS, but Docker Swarm, Hashicorp Nomad, AWS Fargate/ECS, and AWS Lambda have also been developed by the community. Providers are built with the [faas-provider SDK](https://github.com/openfaas/faas-provider).

See also: [Gateway architecture](/architecture/gateway/), [Autoscaling](/architecture/autoscaling/), [Asynchronous invocations](/reference/async/), and [Going to production](/architecture/production/)
