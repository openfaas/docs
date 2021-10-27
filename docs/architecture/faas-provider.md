# `faas-provider` Interface

The `faas-provider` provides the CRUD API for Functions as well as the invoke capabilities.

The [`faas-provider`](https://github.com/openfaas/faas-provider) is an SDK written in Go which conforms to the OpenFaaS Provider's HTTP REST API. Providers which implement the interface should be compatible with the OpenFaaS toolchain and ecosystem, including the UI, CLI, Function Store and Template Store.

![Conceptual diagram](/images/providers/providers-conceptual-flow.png)

Each provider implements the following behaviours:

* CRUD for Functions (or microservices)
* Invocation of Functions via a proxy
* Scaling of Functions
* CRUD for Secrets (optional)
* Log streaming (optional)

## Official Providers

### Kubernetes Provider

[`faas-netes`](https://github.com/openfaas/faas-netes) is the official OpenFaaS Provider for [Kubernetes](https://kubernetes.io/) bundled with the Helm chart. It also provides a Function CRD using the `-operator=true` flag.

### faasd Provider

[`faasd`](https://github.com/openfaas/faasd) OpenFaaS for a single node, without clustering which makes it cost effective and easy to use. Management is simple, it can scale from zero immediately and is useful for appliances and small services. 

### Docker Swarm Provider (deprecated)

[`faas-swarm`](https://github.com/openfaas/faas-swarm) was the official OpenFaaS Provider for [Docker Swarm](https://docs.docker.com/swarm/overview/), which is now deprecated and no longer maintained.

### faas-memory (example)

The [`faas-memory`](https://github.com/openfaas-incubator/faas-memory) provider uses an in-memory store for state and is provided for for testing purposes and and as an example.

## Community Providers

View a list of [Community Providers here](https://github.com/openfaas/faas/blob/master/community.md#openfaas-providers)
