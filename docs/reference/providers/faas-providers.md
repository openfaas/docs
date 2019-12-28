# `faas-provider` Interface
The `faas-provider` provides the CRUD API for Functions as well as the invoke capabilities.

The OpenFaaS team builds and uses the [`faas-provider`](https://github.com/openfaas/faas-provider) project to define the core interface and standard utilities of an OpenFaaS provider.  This interface, written in Go, ensures that you can create a provider which is compliant and compatible with the OpenFaaS gateway.


## Official Providers

### faas-memory
[`faas-memory`](https://github.com/openfaas-incubator/faas-memory) uses in-memory objects to store state and is provided as an official example implementation and for testing purposes.

### faas-netes
[`faas-netes`](https://github.com/openfaas/faas-netes) is the official OpenFaaS provider for [Kubernetes](https://kubernetes.io/).

### faas-swarm
[`faas-swarm`](https://github.com/openfaas/faas-swarm) is the official OpenFaaS provider for [Docker Swarm](https://docs.docker.com/swarm/overview/).

## Community Developed Providers

An active list of community built Providers can be found [here](https://github.com/openfaas/faas/blob/master/community.md#openfaas-providers)


## Going Further

Join `#faas-provider` on [OpenFaaS Slack](https://docs.openfaas.com/community/)
