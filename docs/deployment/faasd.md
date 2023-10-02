# faasd deployment - a lightweight & portable faas engine

faasd is [OpenFaaS](https://github.com/openfaas/) reimagined, but without the cost and complexity of Kubernetes. It runs on a single host with very modest requirements, making it fast and easy to manage. Under the hood it uses [containerd](https://containerd.io/) and [Container Networking Interface (CNI)](https://github.com/containernetworking/cni) along with the same core OpenFaaS components from the main project.

## When should you use faasd over OpenFaaS on Kubernetes?

* You have a cost sensitive project - run faasd on a 5-10 USD VPS or on your Raspberry Pi
* When you just need a few functions or microservices, without the cost of a cluster
* When you don't have the bandwidth to learn or manage Kubernetes
* To deploy embedded apps in IoT and edge use-cases
* To shrink-wrap applications for use with a customer or client

faasd does not create the same maintenance burden you'll find with maintaining, upgrading, and securing a Kubernetes cluster. You can deploy it and walk away, in the worst case, just deploy a new VM and deploy your functions again.

Watch a video comparing the two:

[![](https://img.youtube.com/vi/ZnZJXI377ak/hqdefault.jpg)](https://www.youtube.com/watch?v=ZnZJXI377ak)

[Meet faasd. Look Ma’ No Kubernetes! - Serverless Summit](https://www.youtube.com/watch?v=ZnZJXI377ak)

## Deployment

faasd is a static binary which requires a Linux system configured with systemd.

Minimal system resources are required such as:

* 512MB-1GB RAM (1GB recommended)
* 2-4 vCPU cores minimum
* 10-25GB of disk space
* *x86_64* or *64-bit Arm* CPU architecture
* The Raspberry Pi 3, 4 & 5 are *also supported*

Local development/testing:

* Windows and Mac users can use multipass to deploy faasd in a VM
* Linux users can deploy faasd directly, or also use multipass

Production use:

* Use the bash installation script
* Deploy using the provided cloud-init script
* Deploy using terraform, or adapt the terraform examples

### Get started

* Checkout the [faasd course & handbook](https://gumroad.com/l/serverless-for-everyone-else)
* Browse the [faasd code](https://github.com/openfaas/faasd/) on GitHubs
