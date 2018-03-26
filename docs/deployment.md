# Deployment

> Note: The best place to get an overview of OpenFaaS is in the [Overview page](./).

### A foreword on security

These instructions are for a development environment. If you plan to expose OpenFaaS on the public Internet you need to enable basic authentication with a proxy such as Kong or Traefik at a minimum. TLS is also highly recommended and freely available with LetsEncrypt.org. [Kong guide](https://github.com/openfaas/faas/blob/master/guide/kong_integration.md) [Traefik guide](https://github.com/openfaas/faas/blob/master/guide/traefik_integration.md).

## Kubernetes

OpenFaaS is Kubernetes native and you can follow the [deployment guide here](/deployment/kubernetes/).

## Docker Swarm

You can follow the [deployment guide for Docker Swarm](/deployment/swarm/) or use the Docker Playground below if you don't have direct access to Docker.

### Docker Playground

You can quickly start OpenFaaS on Docker Swarm online using the community-run Docker playground: play-with-docker.com (PWD) by clicking the button below:

[![Try in PWD](https://cdn.rawgit.com/play-with-docker/stacks/cff22438/assets/images/button.png)](http://play-with-docker.com?stack=https://raw.githubusercontent.com/openfaas/faas/master/docker-compose.yml&stack_name=func)
