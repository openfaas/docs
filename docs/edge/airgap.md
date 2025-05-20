# Air Gap with OpenFaaS Edge

OpenFaaS Edge can be installed within an air-gapped environment using images copied from a machine with Internet access, to one without it.

* Download the various images for OpenFaaS Edge, its installer, and any functions you want to a machine with Internet access.
* Transfer the artifacts to the air-gapped machine.
* Decide whether to restore the images to a self-hosted registry on the machine, to the containerd library, or a remote registry available to the air gap.
* Copy across the license file and run the `faasd install` command along with any pull policy and DNS settings you require.

Note:

* If your registry requires authentication, you'll have to create a config file for the credentials. Follow the chapter entitled "Private registries" in Serverless For Everyone Else.
* It is possible to run a self-hosted registry with a self-signed certificate directly on the host with systemd, follow the chapter entitled "Adding a self-hosted container registry" in Serverless For Everyone Else.

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

If you wish to obtain a specific version of the RPM, update the tag from `:latest` to i.e. `:0.2.18`. Browse available versions via `crane ls ghcr.io/openfaasltd/faasd-pro-rpm`.

Then copy all `openfaas-edge-*.rpm` files to the air-gapped machine, and run:

Before installing OpenFaaS Edge ensure all other required packages are installed on the air-gapped system:

```sh
sudo dnf install runc iptables-services
```

Then install the OpenFaaS Edge RPM package:

```bash
sudo dnf install openfaas-edge-*.rpm
```

The command will let you know whether any other required system package are missing such as `selinux-policy`, `libselinux-utils`, `protobuf-c`, and `container-selinux`.

After the installation completes add you OpenFaaS Edge license:

```sh
sudo mkdir -p /var/lib/faasd/secrets
sudo nano /var/lib/faasd/secrets/openfaas_license
```

The final installation step sets up and starts the faasd and faasd-provider services.

If you have a custom DNS server available, specify it using the `--dns-server` flag:

```sh
--dns-server 10.0.0.1
```

If there is no DNS available, you can point faasd at the local host to use systemd-resolved:

```sh
--dns-server 127.0.0.53
```

If your images are restored to the containerd library, you will have to use the `--pull-policy=IfNotPresent` flag to prevent faasd from trying to pull the images from the Internet.

```sh
--pull-policy=IfNotPresent
```

Finally, construct the command to install OpenFaaS Edge:

Example with no DNS server, and images restored to the containerd library:

```sh
sudo /usr/local/bin/faasd install \
  --dns-server 127.0.0.53 \
  --pull-policy=IfNotPresent
```

Example with custom DNS server, and a remote registry:

```sh
sudo /usr/local/bin/faasd install \
  --dns-server 10.0.0.1
```
