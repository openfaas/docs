# faasd deployment - a lightweight & portable faas engine

faasd is [OpenFaaS](https://github.com/openfaas/) reimagined, but without the cost and complexity of Kubernetes. It runs on a single host with very modest requirements, making it fast and easy to manage. Under the hood it uses [containerd](https://containerd.io/) and [Container Networking Interface (CNI)](https://github.com/containernetworking/cni) along with the same core OpenFaaS components from the main project.

This page covers faasd CE and OpenFaaS Edge (a commercial distribution of faasd).

Minimal system resources are required such as:

* 512MB-1GB RAM (1GB recommended)
* 2-4 vCPU cores minimum
* 10-25GB of disk space
* *x86_64* or *64-bit Arm* CPU architecture
* The Raspberry Pi 3, 4 & 5 are *also supported*

Local development/testing:

* Windows and Mac users can use multipass to deploy faasd in a VM
* Linux users can deploy faasd directly, or also use multipass

## When should you use faasd over OpenFaaS on Kubernetes?

* You have a cost sensitive project - run faasd on a 5-10 USD VPS or on your Raspberry Pi
* When you just need a few functions or microservices, without the cost of a cluster
* When you don't have the bandwidth to learn or manage Kubernetes
* To deploy embedded apps in IoT and edge use-cases
* To shrink-wrap applications for use with a customer or client

faasd does not create the same maintenance burden you'll find with maintaining, upgrading, and securing a Kubernetes cluster. You can deploy it and walk away, in the worst case, just deploy a new VM and deploy your functions again.

Watch a video comparing the two:

[![Thumbnail](https://img.youtube.com/vi/ZnZJXI377ak/hqdefault.jpg)](https://www.youtube.com/watch?v=ZnZJXI377ak)

[Meet faasd. Look Ma’ No Kubernetes! - Serverless Summit](https://www.youtube.com/watch?v=ZnZJXI377ak)

## faasd CE - for Personal Use and Small Business Environments

faasd CE is a static binary which requires a Linux system configured with systemd. faasd CE can be used for Personal Use and within Small Business Environments, with a limit of 15 functions and a single namespace. The [faasd EULA](https://github.com/openfaas/faasd/blob/master/EULA.md) explains the terms of use.

### OpenFaaS Edge - for production and resale

OpenFaaS Edge is a commercial distribution of faasd, with enhancements and additional features from [OpenFaaS Pro](/openfaas-pro/introduction). The [OpenFaaS Pro EULA applies](https://github.com/openfaas/faas/blob/master/pro/EULA.md).

* Upgraded Pro components from OpenFaaS Standard: Gateway, Cron Connector, JetStream Queue Worker and Classic Scale to Zero
* Deploy up to 250 functions per installation
* Configure private DNS servers
* Airgap-friendly with installation bundled in an OCI image and rpm package
* Multiple namespace support

This version is intended for resale as part of a wider solution, and to be deployed both into industrial and on-premises environments.

Individual [GitHub Sponsors of OpenFaaS](https://github.com/sponsors/openfaas) (25 USD / mo and higher) can use OpenFaaS Edge for personal use.

When deploying to production, or for customers:

* Use the bash installation script
* Deploy using the provided cloud-init script
* Deploy using terraform, or adapt the terraform examples
* Bundle it as part of a VM image

[Deploy OpenFaaS Edge](https://github.com/openfaas/faasd?tab=readme-ov-file#deploy-openfaas-edge-commercial-distribution-of-faasd)

### Get started with either version

* Checkout the [faasd course & handbook](http://store.openfaas.com/l/serverless-for-everyone-else)
* Browse the [faasd CE repository](https://github.com/openfaas/faasd/) on GitHubs
