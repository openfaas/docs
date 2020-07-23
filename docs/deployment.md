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

## faasd - Serverless for everyone else

faasd is OpenFaaS, reimagined without the complexity and cost of Kubernetes. It runs well on a single host with very modest requirements, and can be deployed in a few moments. Under the hood it uses [containerd](https://containerd.io/) and [Container Networking Interface (CNI)](https://github.com/containernetworking/cni) along with the same core OpenFaaS components from the main project.

When should you use faasd over OpenFaaS on Kubernetes?

* You have a cost sensitive project - run faasd on a 5-10 USD VPS or on your Raspberry Pi
* When you just need a few functions or microservices, without the cost of a cluster
* When you don't have the bandwidth to learn or manage Kubernetes
* To deploy embedded apps in IoT and edge use-cases
* To shrink-wrap applications for use with a customer or client

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
