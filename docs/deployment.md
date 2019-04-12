# Deployment

!!! warning "A foreword on security"
    Make sure that you enable basic authentication if you are exposing OpenFaaS to the public Internet. This will prevent unauthorized access to the OpenFaaS API. It is also highly recommended that you set up TLS with certficates available for free from LetsEncrypt.org.

OpenFaaS can be deployed to Kubernetes and Docker Swarm. We recommend Kubernetes for moving to production, but Docker Swarm can provide a simpler alternative, especially for local development. Functions and microservices built or adapted for OpenFaaS can work with either orchestration platform without changes.

## Kubernetes (recommended for production)

Get started with OpenFaaS on Kubernetes with `helm` (recommended) or plain YAML files. [Deploy to Kubernetes](/deployment/kubernetes/) now.

### OpenShift

OpenShift is a variant of Kubernetes produced by RedHat: [Deploy to OpenShift](/deployment/openshift/)

## Docker Swarm (recommended for learning & starting-out)

If you prefer to use Docker Swarm, then follow the [deployment guide Docker Swarm](/deployment/docker-swarm/).

### Docker Playground

If you cannot run Docker on your local machine, or can't install anything locally, then you can try OpenFaaS on play-with-docker.com (PWD). Follow the [Play-with-Docker guide](/deployment/play-with-docker/) here.
