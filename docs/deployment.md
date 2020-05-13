# Deployment

OpenFaaS can be deployed to a variety of container orchestrators such as Kubernetes, OpenShift, Docker Swarm or to a single host with faasd.

!!! info "Kubernetes or Swarm?"
    We recommend using Kubernetes or OpenShift with OpenFaaS. Whilst Docker Swarm will remain an option for the time-being, the ecosystem for Kubernetes is much richer and commercial support is available from OpenFaaS Ltd.

## PLONK Stack

[PLONK](https://blog.alexellis.io/getting-started-with-the-plonk-stack-and-serverless/) is a Cloud Native stack for building applications which stands for:

* [Prometheus](https://prometheus.io/) - metrics and time-series
* Linux/Linkerd* - OS or service mesh (Linkerd is optional)
* OpenFaaS - management and auto-scaling of compute - PaaS/FaaS, a developer-friendly abstraction on top of Kubernetes. Each function or microservice is built as an immutable Docker container or OCI-format image.
* [NATS](https://nats.io/) - asynchronous message bus / queue
* Kubernetes - declarative, extensible, scale-out, self-healing clustering

OpenFaaS on Kubernetes bundles NATS and Prometheus. You can read about the stack in the [OpenFaaS architecture docs](https://docs.openfaas.com/architecture/stack/#layers-and-responsibilities)

## Kubernetes (recommended for production and for work)

!!! warning "A foreword on security"
    Authentication is enabled by default with OpenFaaS, however you will also need to obtain a TLS certificate for your cluster if you are using OpenFaaS on the public Internet. Free certificates are available from LetsEncrypt.org.

There are three recommended ways to install OpenFaaS to a Kubernetes cluster:

* Using our CLI installer [arkade](https://get-arkade.dev/) - (recommended)
* With the Helm chart, Flux or ArgoCD (GitOps workflow)
* Or using the statically generated YAML files (not recommended)

Find out more about each option and how to deploy OpenFaaS to Kubernetes:

[Deploy to Kubernetes](/deployment/kubernetes/)

## faasd with containerd

faasd is a light-weight option for adopting OpenFaaS which uses the same tooling, ecosystem, templates, and containers as OpenFaaS on Kubernetes. The difference is that it runs on a single host removing the need for complex infrastructure and maintenance.

faasd is built with [containerd](https://containerd.io/) and the [Container Networking Interface (CNI)](https://github.com/containernetworking/cni) project.

Why might you try faasd?

* As a lightweight option it is well-suited to use-cases such as: appliances, VMs, embedded use, edge, and for IoT. 
* Teams may also find faasd useful for local development before adopting or deploying to Kubernetes.
* Teams who feel that they could benefit from functions and microservices, but who do not have the bandwidth to learn Kubernetes and manage a cluster

[Deploy faasd](https://github.com/openfaas/faasd/)

## OpenShift

OpenShift is a variant of Kubernetes produced by RedHat.

You can deploy to OpenShift using our CLI installer <a href="https://get-arkade.dev/">arkade</a> or with the standard helm chart.

[Deploy to OpenShift](/deployment/openshift/)

## Docker Swarm

!!! warning "A foreword on security"
    Bear in mind that Docker Swarm is widely consider to be approaching "End of Life". Kubernetes with <a href="https://k3s.io">k3s.io</a>, or faasd (above) may make a better alternative for you and your team.

If you prefer to use Docker Swarm, then follow the [deployment guide Docker Swarm](/deployment/docker-swarm/).

### Docker Swarm Playground

If you cannot run Docker on your local machine, or can't install anything locally, then you can try OpenFaaS on play-with-docker.com (PWD). Follow the [playground guide here](/deployment/play-with-docker/)
