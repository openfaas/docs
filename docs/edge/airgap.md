# Air Gap with OpenFaaS Edge

OpenFaaS Edge can be installed within an air-gapped environment using images copied from a machine with Internet access, to one without it.

## Download images for offline usage

You can download, transfer and restore the images whichever way you prefer, however we maintain a dedicated, supported tool to do this for you called airfaas.

### Download the OpenFaaS Edge images for offline access

On a PC with Internet access, run the following command to download the images:

```bash
faas-cli plugin get airfaas
```

Now, download the images for OpenFaaS Edge from your docker-compose.yaml file:

```bash
faas-cli airfaas download images \
  --yaml ./docker-compose.yaml \
  --path ./images \
  openfaas-edge
```

If you do not have a `docker-compose.yaml` file, you can export the installation bundle locally to get it.

```sh
mkdir -p ./faasd-pro
arkade oci install --path ./faasd-pro ghcr.io/openfaasltd/faasd-pro:latest
```
The docker-compose.yaml file can be found at `./faasd-pro/var/lib/faasd/docker-compose.yaml`

### Download your functions

You can also download your functions using the same method.

```bash
faas-cli airfaas download images \
  --yaml ./stack.yaml \
  --path ./images \
  functions
```

## Transfer the images

Transfer the `./images` directory to the air-gapped machine using your preferred method. This could be a USB drive, SCP, rsync, or any other method you prefer.

## Restore the images

When running OpenFaaS Edge in an air-gap, you can restore the images to either a local registry or the containerd library.

### Restore images to the containerd library

The easiest way to test an air-gapped installation, is to bypass the need for a local registry, and to restore the images directly to the containerd library.

OpenFaaS Edge binaries and dependencies need to be installed before you can run the `restore` command. Follow the [air-gapped installation instructions](#perform-the-installation) and restore the images right before running `faasd install`.

Restore the OpenFaaS Edge images:

```bash
faas-cli airfaas restore \
  --containerd \
  --namespace openfaas \
  ./images/openfaas-edge/images.json
```

If you need to restore any of your own functions, make sure you pass the `--namespace` flag, i.e.

```bash
faas-cli airfaas restore \
  --containerd \
  --namespace openfaas-fn \
  ./images/functions/images.json
```

### Restore images to a local registry

You can restore the images to a local registry using the following command:

```bash
faas-cli airfaas restore \
  --path ./images/openfaas-edge/images.json
```

To update the original registry references i.e. `ghcr.io/openfaasltd` to your own i.e. `localhost:5000/openfaasltd`, you can use the `--prefix` flag.

When using a self-signed certificate, use the `--insecure-registry` flag to skip TLS verification.

Further examples are available via the `--help` flag.

## Perform the installation

### Debian-based systems

If you're using an Operating System such as Ubuntu, you can export the installation bundle and copy it to the air-gapped machine, then perform the installation as normal.

Ensure required packages are installed on the air-gapped system:

```sh
sudo apt-get install runc bridge-utils iptables iptables-persistent
```

Download the installation package:

```bash
mkdir -p ./faasd-pro
arkade oci install --path ./faasd-pro ghcr.io/openfaasltd/faasd-pro:latest
```

Then copy the `faasd-pro` directory to the air-gapped machine.

Run the install script on the remote server:

```bash
sudo -E ./faasd-pro/install.sh ./faasd-pro/
```

After the installation script completes add you OpenFaaS Edge license:

```sh
sudo mkdir -p /var/lib/faasd/secrets
sudo nano /var/lib/faasd/secrets/openfaas_license
```

Perform the final installation step:

```sh
sudo -E sh -c "cd ./faasd-pro/var/lib/faasd && faasd install"
```

By default OpenFaaS uses Google's public DNS servers you need to specify custom DNS servers during the installation phase by setting the `--dns-server` flag:

```sh
sudo faasd install --dns-server 127.0.0.53
```

Make sure to also add `--pull-policy=IfNotPresent` when images were restored directly to the containerd library. This is not required when using a local image registry.

### RHEL-like systems

For Operating Systems such as Oracle Linux, Alma Linux, and Rocky Linux you can use our official rpm package to install OpenFaaS Edge.

Download it on a machine with Internet access, transfer it to the air-gapped machine, and install it using:

```bash
arkade oci install --path . ghcr.io/openfaasltd/faasd-pro-rpm:latest
```

Before installing OpenFaaS Edge ensure all other required packages are installed on the air-gapped system:

```sh
sudo dnf install runc iptables-services
```

Then copy all .rpm files to the air-gapped machine, and run:

```bash
dnf install openfaas-edge-*.rpm
```

After the installation completes add you OpenFaaS Edge license:

```sh
sudo mkdir -p /var/lib/faasd/secrets
sudo nano /var/lib/faasd/secrets/openfaas_license
```

Perform the final installation step:

```sh
sudo /usr/local/bin/faasd install
```

By default OpenFaaS uses Google's public DNS servers you need to specify custom DNS servers during the installation phase by setting the `--dns-server` flag:

```sh
sudo /usr/local/bin/faasd install --dns-server 127.0.0.53
```

Make sure to also add `--pull-policy=IfNotPresent` when images were restored directly to the containerd library. This is not required when using a local image registry.

It is possible to specify a different version of the package by changing the `latest` tag to a specific version, e.g. `v0.2.16`.

Versions can be inspected using the crane tool and `crane ls ghcr.io/openfaasltd/faasd-pro-rpm`.

