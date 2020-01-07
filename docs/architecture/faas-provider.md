# `faas-provider` Interface

The `faas-provider` provides the CRUD API for Functions as well as the invoke capabilities.

The [`faas-provider`](https://github.com/openfaas/faas-provider) is an SDK written in Go which conforms to the OpenFaaS provider's HTTP REST API. Providers which implement the interface should be compatible with the OpenFaaS toolchain and ecosystem, including the UI, CLI, Function Store, Template Store and OpenFaaS Cloud.

![Conceptual diagram](/images/providers/providers-conceptual-flow.png)

Each provider implements the following behaviours:

* CRUD for Functions (or microservices)
* Invocation of Functions via a proxy
* Scaling of Functions
* CRUD for Secrets (optional)
* Log streaming (optional)

## Official Providers

### Kubernetes Providers

[`faas-netes`](https://github.com/openfaas/faas-netes) is the official OpenFaaS provider for [Kubernetes](https://kubernetes.io/) bundled with the Helm chart.

[`openfaas-operator`](https://github.com/openfaas-incubator/openfaas-operator) a [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) which also provides a CRD, also available as an option in the helm chart.

### Docker Swarm Provider

[`faas-swarm`](https://github.com/openfaas/faas-swarm) is the official OpenFaaS provider for [Docker Swarm](https://docs.docker.com/swarm/overview/).

### faas-memory

The [`faas-memory`](https://github.com/openfaas-incubator/faas-memory) provider uses an in-memory store for state and is provided for for testing purposes and and as an example.

## Community Providers

View a list of [Community Providers here](https://github.com/openfaas/faas/blob/master/community.md#openfaas-providers)

## Discussion and getting more information

Join the `#faas-provider` channel on [OpenFaaS Slack](https://docs.openfaas.com/community/)
