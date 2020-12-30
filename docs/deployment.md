# Deployment

OpenFaaS can be deployed to a variety of container orchestrators such as Kubernetes, OpenShift, Docker Swarm or to a single host with faasd.

!!! info "Kubernetes or faasd?"
    We recommend using Kubernetes or OpenShift when using OpenFaaS for work because it can scale well, and OpenFaaS Ltd can provide commercial support, should you need it. faasd is already being used in production by some companies, but you should make yourself aware of the tradeoffs. Users can move between either deployment option at a later date.

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

## Docker Swarm (deprecated)

!!! warning "Deprecation of free community support"
    Free community support for Docker Swarm has been discontinued. If you run Docker Swarm and OpenFaaS in your business and require continued support, then [contact support](https://openfaas.com/support/) as soon as you can.

Why has this decision been made? Docker Swarm is widely consider to be "End of Life" product. Projects like Kubernetes have a much broader ecosystem and are receiving continual investment from their respective communities.

If you were attracted to Docker Swarm because you believe it to be easier to manage than Kubernetes, we recommend taking a look at a managed Kubernetes service, or [k3s.io](https://k3s.io).

If you need next to no operations, then faasd (above) should be the top of your list.
