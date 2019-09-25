## OpenFaaS stack


### Layers in OpenFaaS

The recommended platform for deploying OpenFaaS is Kubernetes, whether that is a local environment, a self-hosted cluster, or with a managed service such as AWS Elastic Kubernetes Service (EKS).

![Layers](https://github.com/openfaas/faas/raw/master/docs/of-layer-overview.png)

* OpenFaaS Cloud builds upon OpenFaaS to deliver GitOps with GitHub.com or GitLab self-hosted
* NATS provides asynchronous execution and queuing
* Prometheus provides metrics and enables auto-scaling through AlertManager
* A container registry holds each immutable artifact that can be deployed on OpenFaaS via the API

### Workflow

![Workflow](https://github.com/openfaas/faas/blob/master/docs/of-workflow.png?raw=true)

The Gateway can be accessed through its REST API, via the CLI or through the UI. All services or functions get a default route exposed, but custom domains can also be used for each endpoint.

Prometheus collects metrics which are available via the Gateway's API and which are used for auto-scaling.

By changing the URL for a function from `/function/` to `/async-function/` an invocation can be run in a queue using NATS Streaming.

faas-netes is the most popular orchestration provider for OpenFaaS, but Docker Swarm, Hashicorp Nomad, AWS Fargate/ECS, and AWS Lambda are also available.

See also: [Gateway architecture](https://docs.openfaas.com/architecture/gateway/), [Autoscaling](https://docs.openfaas.com/architecture/autoscaling/), and [Going to production](https://docs.openfaas.com/architecture/production/)
