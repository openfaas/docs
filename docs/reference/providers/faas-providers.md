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

### faas-containerd
[`faas-containerd`](https://github.com/alexellis/faas-containerd), by Alex Ellis, is an OpenFaaS provider for [containerd](https://containerd.io/) - single node / edge workloads.

### faas-nomad
[`faas-nomad`](https://github.com/hashicorp/faas-nomad), by Andrew Cornies & Nic Jackson (Hashicorp), is an OpenFaas Provider for the [Nomad](https://www.nomadproject.io/) platform.

### faas-fargate
[`faas-fargate`](https://github.com/ewilde/faas-fargate), by Edward Wilde, is an OpenFaaS provider for [AWS Fargate](https://aws.amazon.com/fargate/)


### faas-lambda
`faas-lambda`, by Ed Wilde and Alex Ellis, is an OpenFaaS provider for [AWS Lambda](https://aws.amazon.com/lambda/). More details via [sales@openfaas.com](mailto:sales@openfaas.com)

### faas-federation
[`faas-federation`](https://github.com/openfaas-incubator/faas-federation), by Ed Wilde and Alex Ellis,  is federation provider to route between one or more providers.


## Going Further

Join `#faas-provider` on [OpenFaaS Slack](https://docs.openfaas.com/community/)
