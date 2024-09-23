# Deployment

OpenFaaS can be deployed to a variety of container orchestrators such as Kubernetes, K3s, OpenShift, or to a single host with faasd.

## OpenFaaS Pro - Standard or for Enterprises (production, commercial use)

OpenFaaS Pro is a commercially supported version of OpenFaaS which is designed and licensed for production use. The two variants are:

* OpenFaaS Standard - intended for single teams, single-tenanted environments
* OpenFaaS for Enterprises - intended for larger organisations or SaaS vendors wanting to integrate functions into their product

* [Deploy OpenFaaS Pro - Standard or for Enterprises](/deployment/pro)

See also:

* [High level comparison and contact form](https://openfaas.com/pricing)
* [Detailed comparison of editions](https://docs.openfaas.com/openfaas-pro/introduction/)

## OpenFaaS Community Edition (CE) for Kubernetes

The Community Edition of OpenFaaS is intended for personal use only, and has a [60-day limit for commercial use](https://github.com/openfaas/faas/blob/master/EULA.md).

!!! warning "A foreword on security"
    Authentication is enabled by default with OpenFaaS, however you will also need to obtain a TLS certificate for your cluster if you are using OpenFaaS on the public Internet. Free certificates are available from LetsEncrypt.org.

There are three recommended ways to install OpenFaaS to a Kubernetes cluster:

* Using our CLI installer [arkade](https://arkade.dev/) - (recommended)
* With the Helm chart, Flux or ArgoCD (GitOps workflow)
* Or using the statically generated YAML files (not recommended)

Find out more about each option and how to deploy OpenFaaS to Kubernetes:

[Deploy to Kubernetes](/deployment/kubernetes/)

## faasd - Serverless for everyone else

faasd is OpenFaaS, reimagined without the complexity and cost of Kubernetes. It runs well on a single host with very modest requirements, and is largely API-compatible with OpenFaaS on Kubernetes. Under the hood it uses [containerd](https://containerd.io/) and [Container Networking Interface (CNI)](https://github.com/containernetworking/cni) along with the same core components from OpenFaaS CE.

When should you use faasd over OpenFaaS CE on Kubernetes?

* You're building automation/glue-code, web portals, cron jobs, bots, or webhook receivers.
* You want to be able to run your code locally, using containers.
* You want a way to do remote deployments over a REST API.
* You either don't have the bandwidth to manage or learn Kubernetes.
* You don't need planet scale.

You may also be interested in:

* Deploying to constrained IoT or edge devices
* Shrink-wrapping functions into a VM image to deploy for customers or clients.

faasd CE is available for personal use, and limited commercial use in a "Small Business Environment", [read the EULA to find out if you quality](https://github.com/openfaas/faasd/blob/master/EULA.md).

The CE version is limited to a single namespace, with a limit of 15 functions, which should be sufficient for most personal projects and for internal use within a Small Business Environment. OpenFaaS Edge is a commercial distribution of faasd with additional features and higher limits.

[Deploy faasd CE](https://github.com/openfaas/faasd/)

### OpenFaaS Edge

OpenFaaS Edge is a commercial distribution of faasd, with enhancements and additional features from [OpenFaaS Pro](/openfaas-pro/introduction). The [OpenFaaS Pro EULA applies](https://github.com/openfaas/faas/blob/master/pro/EULA.md).

* Upgraded Cron Connector, JetStream Queue Worker and Classic Scale to Zero from OpenFaaS Pro
* Deploy up to 250 functions
* Configure private DNS servers
* Airgap-friendly with installation bundled in an OCI image
* Multiple namespace support

This version is intended for resale as part of a wider solution, and to be deployed both into industrial and on-premises environments.

Individual [GitHub Sponsors of OpenFaaS](https://github.com/sponsors/openfaas) (25 USD / mo and higher) can use OpenFaaS Edge for personal use.

[Deploy OpenFaaS Edge](https://github.com/openfaas/faasd?tab=readme-ov-file#deploy-openfaas-edge-commercial-distribution-of-faasd)

## OpenShift - 3.x / 4.x

[OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift) is a variant of Kubernetes produced by Red Hat.

You can deploy to OpenShift using the standard Helm chart as per the Kubernetes instructions, or with <a href="https://arkade.dev/">arkade</a>.

[Deploy to OpenShift](/deployment/openshift/)

