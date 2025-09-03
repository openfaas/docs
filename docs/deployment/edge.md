## OpenFaaS Edge

OpenFaaS Edge (aka faasd-pro) is the commercial distribution of our faasd project.

faasd takes the core OpenFaaS components, and packages them for use on a single host, without the cost and complexity of Kubernetes. It is distributed as a single binary, and runs as a systemd service. It can manage both functions, and a set of stateful services such as databases, messages brokers, monitoring tools, and custom agents.

Kubernetes can be a complex system to install and manage, needing many add-on components to be installed, configured and maintained over time. Each version of Kubernetes can introduce breaking changes. faasd presents a simpler alternative, where clustering and scaling are traded-off in return for a stable system, that can be deployed as an appliance and largely left alone.

faasd combines the following battle-tested components from OpenFaaS and Kubernetes:

* The Core Components of OpenFaaS - the gateway, the queue-worker, Prometheus monitoring
* [containerd](https://containerd.io/) - the container runtime used in Kubernetes
* [Container Networking Interface (CNI)](https://github.com/containernetworking/cni) - the networking interface used in Kubernetes

Minimum system requirements:

* *x86_64* or *64-bit Arm* CPU architecture
* 512MB-1GB RAM (1GB recommended)
* 2-4 vCPU
* 10-25GB disk space
* The Raspberry Pi 3, 4 & 5 are *also supported*

## Getting started (faasd CE or faasd-pro)

If you want to use faasd for non-commercial, personal use, faasd CE is available for you for free. Up to 15 functions are supported, including private registries and a single namespace. If you need to use faasd for commercial use, you can evaluate a single installation of faasd CE for a period of 60 days.

After that time, you will need to purchase a license for OpenFaaS Edge (faasd-pro).

If you need to use faasd at work, on work time, for a commercial product or service, or for an end-client, then you will need to purchase a license for OpenFaaS Edge (faasd-pro).

* Upgraded Pro components from OpenFaaS Standard: Gateway, Cron Connector, JetStream Queue Worker and Classic Scale to Zero
* Deploy up to 250 functions per installation*
* Configure private DNS servers
* Airgap-friendly with installation bundled in an OCI image or RPM package
* Detailed RAM/CPU metrics for stateful containers, and functions, including Out Of Memory (OOM) events
* Multiple namespace support
* Network policies/segmentation to secure multi-tenant environments

`*` The number of functions is tied to your license, with a maximum of 250 functions per installation.

This version is intended for resale as part of a wider solution, and to be deployed both into industrial and on-premises environments.

## Prerequisites

The quickest and easiest option for trying out faasd is to use a Linux VM from a cloud provider. You can typically create these in a minute or two, and then delete them when you're finished.

If you want to try out faasd on your Mac or Windows machine, then we recommend [Multipass](https://multipass.run/) by Canonical. Multipass is a tool that allows you to create a Linux VM on your Mac or Windows machine.

faasd must not be co-located with Docker, since they both make use of containerd, iptables and CNI, and will clash with one-another. It is designed to be an appliance, and have sole tenancy of any host.

Images should always be built on another host, and pushed to a registry that faasd can subsequently pull from.

A utility script is provided for either version, which can also be run via cloud-init for automated and production deployments.

## OpenFaaS Edge/faasd-pro (commercial use)

Everything required for faasd-pro is packaged in an container image, which makes for easy off-line installation. An RPM package is also available for RHEL-like Operating Systems.

```bash
curl -sLSf \
    https://raw.githubusercontent.com/openfaas/faasd/refs/heads/master/hack/install-edge.sh \
    -o install-edge.sh && \
chmod +x install-edge.sh
sudo -E ./install-edge.sh
```

!!! Note "Offline installation"

    For an offline installation see: [Air-gapped OpenFaaS Edge](/edge/airgap/)

### OpenFaaS Edge on RHEL-like systems

For Operating Systems such as Oracle Linux, Alma Linux, and Rocky Linux you can use our official rpm package to install OpenFaaS Edge.

The rpm package is published to a container registry:

```bash
arkade oci install --path . ghcr.io/openfaasltd/faasd-pro-rpm:latest
```

To download a specific version of the rpm, update the tag from `:latest` to i.e. `:0.2.18`. Browse available versions via `crane ls ghcr.io/openfaasltd/faasd-pro-rpm`.

Then install using the rpm package:

```bash
sudo dnf install openfaas-edge-*.rpm
```

Note: additional packages may be required such as runc, iptables-services, selinux-policy, libselinux-utils, protobuf-c, and container-selinux.

### Upgrading OpenFaaS Edge

#### Upgrade the OpenFaaS Edge binary

The following will download the latest OpenFaaS Edge binary, and overwrite the one on the system:

```bash
mkdir -p ./staging-area/
arkade oci install --path ./staging-area/ ghcr.io/openfaasltd/faasd-pro:latest
sudo systemctl stop faasd
sudo systemctl stop faasd-provider

sudo cp ./staging-area/usr/local/bin/faasd /usr/local/bin/
```

Then restart both services:

```bash
sudo systemctl restart faasd
sudo systemctl restart faasd-provider
```

You can also perform a re-installation using your preferred method - bash script, RPM/Debian package. In this case, make sure you take a backup of your `docker-compose.yaml` file at `/var/lib/faasd/` in case it gets overwritten by the one shipping in the installation.

#### Upgrade container images

You can use [arkade](https://github.com/alexellis/arkade) to upgrade the container images for the various data-plane components such as the gateway, queue-worker, cron-connector, nats and so forth run the following:

```bash
$ sudo -i

# cd /var/lib/faasd/
# cp ./docker-compose.yaml{,old} 
```

View upgrades without changing the file:

```bash
$ arkade chart upgrade --verbose -f ./docker-compose.yaml

2025/09/03 09:48:17 Verifying images in: ./docker-compose.yaml
2025/09/03 09:48:17 Found 7 images
2025/09/03 09:48:18 [ghcr.io/openfaasltd/gateway] 0.5.0 => 0.5.1
2025/09/03 09:48:18 [ghcr.io/openfaasltd/jetstream-queue-worker] 0.4.0 => 0.4.2
2025/09/03 09:48:18 [ghcr.io/openfaasltd/openfaas-dashboard] 0.5.36 => 0.5.37
2025/09/03 09:48:18 [docker.io/library/nats] 2.11.7 => 2.11.8
```

Write changes to the file:

```bash
$ arkade chart upgrade --verbose --write -f ./docker-compose.yaml
```

Then restart the faasd service:

```bash
sudo systemctl restart faasd
```

## faasd CE (non-commercial use only)

faasd CE supports 15 functions and needs a computer with a stable Internet connection to run. There are restrictions on commercial use, but [individuals](https://github.com/openfaas/faasd/blob/master/EULA.md) can use it for free for personal, non-commercial use.

```bash
git clone https://github.com/openfaas/faasd --depth=1
cd faasd

./hack/install.sh
```

## Fix for DigitalOcean Droplets (VMs)

Droplets on DigitalOcean have journalctl disabled, and use the legacy syslog service instead, this prevents log from being fetched for specific services and functions.

```bash
curl -SsLfL https://raw.githubusercontent.com/openfaas/faasd/refs/heads/master/hack/enable-journal.sh -o enable-journal.sh

# Feel free to read/browse the script before running it
chmod +x enable-journal.sh
sudo ./enable-journal.sh
```

## Documentation and handbook

For either edition, the complete handbook for faasd ["Serverless For Everyone Else"](https://store.openfaas.com/l/serverless-for-everyone-else?layout=profile) is available in the OpenFaaS Store. It contains complete examples for deploying faasd, building functions in Node.js, setting up TLS and custom domains, monitoring, background jobs, and cron schedules.

<a href="https://openfaas.gumroad.com/l/serverless-for-everyone-else">
<img src="https://blog.alexellis.io/content/images/2025/08/serverless.png" width="40%"></a>


Any examples of functions on the blog or in the documentation for OpenFaaS on Kubernetes, will generally work with faasd without modification.

Additional topics covered by the handbook:

* Should you deploy to a VPS or Raspberry Pi?
* Deploying your server with bash, cloud-init or terraform
* Using a private container registry
* Finding functions in the store
* Building your first function with Node.js
* Using environment variables for configuration
* Using secrets from functions, and enabling authentication tokens
* Customising templates
* Monitoring your functions with Grafana and Prometheus
* Scheduling invocations and background jobs
* Tuning timeouts, parallelism, running tasks in the background
* Adding TLS to faasd and custom domains for functions
* Self-hosting on your Raspberry Pi
* Adding a database for storage with InfluxDB and Postgresql
* Troubleshooting and logs
* CI/CD with GitHub Actions and multi-arch
* Taking things further, community and case-studies

See also: [Serverless For Everyone Else eBook](https://store.openfaas.com/l/serverless-for-everyone-else?layout=profile)

### Video overview

Watch the presentation from KubeCon, where faasd was first announced:

[![Thumbnail](https://img.youtube.com/vi/ZnZJXI377ak/hqdefault.jpg)](https://www.youtube.com/watch?v=ZnZJXI377ak)

[Meet faasd. Look Ma’ No Kubernetes! - Serverless Summit](https://www.youtube.com/watch?v=ZnZJXI377ak)

An additional training video and walkthrough is available via the [Serverless For Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else?layout=profile) handbook package.

