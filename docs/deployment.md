# Deployment

OpenFaaS can be deployed to a variety of container platforms such as Kubernetes, OpenShift, Docker Swarm and to containerd using the faasd project.

Whilst support is available for Docker Swarm and faasd, we recommend using Kubernetes in production and for work projects.

## PLONK Stack

[PLONK](https://blog.alexellis.io/getting-started-with-the-plonk-stack-and-serverless/) is a cloud native stack for application developers and an acronym that stands for:

* [Prometheus](https://prometheus.io/) - metrics and time-series
* Linux/Linkerd* - OS or service mesh (Linkerd is optional)
* OpenFaaS - management and auto-scaling of compute - PaaS/FaaS, a developer-friendly abstraction on top of Kubernetes. Each function or microservice is built as an immutable Docker container or OCI-format image.
* [NATS](https://nats.io/) - asynchronous message bus / queue
* Kubernetes - declarative, extensible, scale-out, self-healing clustering

Find out more [in the OpenFaaS architecture](https://docs.openfaas.com/architecture/stack/#layers-and-responsibilities)

## Kubernetes (recommended for production and for work)

!!! warning "A foreword on security"
    Authentication is enabled by default with OpenFaaS, however you will also need to obtain a TLS certificate for your cluster if you are using OpenFaaS on the public Internet. Free certificates are available from LetsEncrypt.org.

There are three recommended ways to install OpenFaaS to a Kubernetes cluster:

* Using the helm chart with our [arkade](https://arkade.dev/) installer, (recommended for dev/test)
* Using the helm chart directly or [via Weave Flux](https://www.openfaas.com/blog/openfaas-flux/) - ideal for a GitOps/IaaC configuration
* Using the plain YAML files - generated YAML which you will need to customise

Start here: [Deploy to Kubernetes](/deployment/kubernetes/)

## faasd with containerd

faasd is a light-weight option for adopting OpenFaaS which uses the same tooling, ecosystem, templates, and containers as OpenFaaS on Kubernetes, but which doesn't require cluster management. faasd uses [containerd](https://containerd.io/) as a runtime and [CNI](https://github.com/containernetworking/cni) for container networking.

Why might you try faasd?

* It's a lightweight option and is suited to use-cases such as: appliances, VMs, embedded use, edge, and for IoT. 
* Teams may also find faasd useful for local development before deploying to Kubernetes.
* Teams who feel that they could benefit from functions and microservices, but who do not have the bandwidth to learn about Kubernetes may prefer this option.

Get started with [faasd](https://github.com/alexellis/faasd/)

## OpenShift

OpenShift is a variant of Kubernetes produced by RedHat: [Deploy to OpenShift](/deployment/openshift/)

## Docker Swarm

If you prefer to use Docker Swarm, then follow the [deployment guide Docker Swarm](/deployment/docker-swarm/).

### Docker Swarm Playground

If you cannot run Docker on your local machine, or can't install anything locally, then you can try OpenFaaS on play-with-docker.com (PWD). Follow the [playground guide here](/deployment/play-with-docker/)
