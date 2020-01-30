# Deployment

OpenFaaS can be deployed to a variety of container platforms such as Kubernetes, OpenShift, Docker Swarm and to containerd using the faasd project.

Whilst support is available for Docker Swarm and faasd, we recommend using Kubernetes in production and for work projects.

## Kubernetes (recommended for production)

!!! warning "A foreword on security"
    Authentication is enabled by default with OpenFaaS, however you will also need to obtain a TLS certificate for your cluster if you are using OpenFaaS on the public Internet. Free certificates are available from LetsEncrypt.org.

There are three recommended ways to install OpenFaaS to a Kubernetes cluster:

* Using the helm chart with our [k3sup (ketchup)](https://k3sup.dev/) installer, (recommended for dev/test)
* Using the helm chart directly or [via Weave Flux](https://www.openfaas.com/blog/openfaas-flux/) - ideal for a GitOps/IaaC configuration
* Using the plain YAML files - generated YAML which you will need to customise

Start here: [Deploy to Kubernetes](/deployment/kubernetes/)

## faasd with containerd

faasd is a new OpenFaaS version which uses the same tooling, ecosystem, templates, and containers as OpenFaaS on Kubernetes, but which doesn't require cluster management. faasd uses containerd as a runtime and CNI for container networking.

It's a lightweight option and is suited to use-cases such as:

- appliances
- VMs
- embedded use
- edge
- and for IoT.

Teams who feel that they could benefit from functions and microservices, but who do not have the bandwidth to learn about Kubernetes may prefer this option.

* Get started with [faasd](https://github.com/alexellis/faasd/)

## OpenShift

OpenShift is a variant of Kubernetes produced by RedHat: [Deploy to OpenShift](/deployment/openshift/)

## Docker Swarm

If you prefer to use Docker Swarm, then follow the [deployment guide Docker Swarm](/deployment/docker-swarm/).

### Docker Swarm Playground

If you cannot run Docker on your local machine, or can't install anything locally, then you can try OpenFaaS on play-with-docker.com (PWD). Follow the [playground guide here](/deployment/play-with-docker/)
