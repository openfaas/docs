# Function Builder API for OpenFaaS Edge

The Function Builder API is available in OpenFaaS Edge and allows you to build functions from source code using a variety of templates.

The builder runs as a non-root user making use of user namespaces in Linux.

## Prerequisites

* An OpenFaaS for Enterprises license or an additional entitlement for the Function Builder API is required to use this feature.
* Your operating system must support user namespaces, generally most modern Linux distributions do.
* Docker must not be installed on the host system.
* faasd-pro version 0.2.26 or later is required.

## Configure a registry

We will be deploying a local container registry as an additional service with faasd and configure the function builder to push images to it.

Create the credentials that will be used to login to the registry. The following commands create credentials for a user named faasd.
The credentials are saved to the file `/var/lib/faasd/registry/auth/htpasswd` in a hashed format, you’ll also need to take a copy of the plaintext version of the password, so that you can authenticate
to the registry.

Ensure `htpasswd`is installed on your system:

```sh
# On Debian run:
sudo apt install apache2-utils

# On RHEL run:
sudo dnf install httpd-tools
```

```sh
export PASSWORD=$(openssl rand -base64 16)
echo $PASSWORD > ~/registry-password.txt

htpasswd -Bbc ./htpasswd faasd $PASSWORD
sudo mkdir -p /var/lib/faasd/registry/auth
sudo mv ~/htpasswd /var/lib/faasd/registry/auth/htpasswd
```

Create a configuration file for the registry:

```sh
sudo tee /var/lib/faasd/registry/config.yml > /dev/null <<EOF
version: 0.1
log:
  accesslog:
    disabled: true
  level: warn
  formatter: text

storage:
  filesystem:
    rootdirectory: /var/lib/registry
auth:
  htpasswd:
    realm: basic-realm
    path: /etc/registry/htpasswd
http:
  addr: 0.0.0.0:5000
  relativeurls: false
  draintimeout: 60s
EOF
```

Create a crednetials file that can be use by faasd and the pro-builder to push and pull images from the registry. The faas-cli has a utility command that can be used to create the credentials file:

```sh
cat ~/registry-password.txt | faas-cli registry-login \
  --server http://registry:5000 \
  --username faasd \
  --password-stdin
```

We are using the `--server` flag to point to the local registry using its internal service name and port.

The file will be created in the `.credentials` folder. Copy the file so that it can be accessed by faasd and the function builder:

```sh
# Ensure faasd-provider can pull images from the faasd service".
sudo mkdir -p /var/lib/faasd/.docker
sudo cp ./credentials/config.json /var/lib/faasd/.docker/config.json
# Ensure the pro-builder can mount the credentials file.
sudo cp ./credentials/config.json /var/lib/faasd/secrets/docker-config
```

Just like the registry the function builder will be running as a faasd service and is able to reach the registry using the internal service name.
To be able to access the registry from the host machine, update the `/etc/hosts` file. This ensures the faasd-provider can also access the registry using the same service name.

```sh
echo "127.0.0.1 registry" | sudo tee -a /etc/hosts
```

Update the faasd-provider service to add the registry as an insecure registry. This is not required if you configure TLS for the registry.

Edit `/lib/systemd/system/faasd-provider.service` and add the flag `--insecure-registry http://registry:5000` to the `ExecStart` command:

```diff
[Unit]
Description=faasd-provider

[Service]
MemoryMax=500M
Environment="secret_mount_path=/var/lib/faasd/secrets"
Environment="basic_auth=true"
Environment="hosts_dir=/var/lib/faasd"
ExecStart=/usr/local/bin/faasd provider \
  --insecure-registry http://registry:5000 \
  --dns-server 8.8.8.8 --dns-server 8.8.4.4 \
  --pull-policy Always \
  --license-file /var/lib/faasd/secrets/openfaas_license \
+ --insecure-registry http://registry:5000
Restart=on-failure
RestartSec=60s
# Keep logging child process running when the main process get killed.
KillMode=process
WorkingDirectory=/var/lib/faasd-provider

[Install]
WantedBy=multi-user.target
```

Make sure to reload the systemd daemon and restart the faasd-provider service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart faasd-provider
```

## Create a payload secret

The payload secret will be used to sign the payloads sent to the Function Builder's API.

```bash
openssl rand -base64 32 | sudo tee /var/lib/faasd/secrets/payload-secret
```

## Update your docker-compose.yaml

Add the following services to your `docker-compose.yaml` file:

```yaml
  registry:
    image: docker.io/library/registry:3
    volumes:
    - type: bind
      source: ./registry/data
      target: /var/lib/registry
    - type: bind
      source: ./registry/auth
      target: /etc/registry/
      read_only: true
    - type: bind
      source: ./registry/config.yml
      target: /etc/docker/registry/config.yml
      read_only: true
    deploy:
      replicas: 1
    ports:
      - "127.0.0.1:5000:5000"

  pro-builder:
    depends_on: [buildkit]
    user: "app"
    group_add: ["1000"]
    restart: always
    image: ghcr.io/openfaasltd/pro-builder:0.5.3
    environment:
      buildkit-workspace: /tmp/
      enable_lchown: false
      insecure: true
      buildkit_url: unix:///home/app/.local/run/buildkit/buildkitd.sock
      disable_hmac: false
#    max_inflight: 10 # Set this line if you wish to limit the amount of concurrent builds
    command:
     - "./pro-builder"
     - "-license-file=/run/secrets/openfaas-license"
    volumes:
      - type: bind
        source: ./secrets/payload-secret
        target: /var/openfaas/secrets/payload-secret
      - type: bind
        source: ./secrets/openfaas_license
        target: /run/secrets/openfaas-license
      - type: bind
        source: ./secrets/docker-config
        target: /home/app/.docker/config.json
      - type: bind
        source: ./buildkit-rootless-run
        target: /home/app/.local/run
      - type: bind
        source: ./buildkit-sock
        target: /home/app/.local/run/buildkit
    deploy:
      replicas: 1
    ports:
     - "127.0.0.1:8088:8080"

  buildkit:
    restart: always
    image: docker.io/moby/buildkit:v0.23.2-rootless
    group_add: ["2000"]
    user: "1000:1000"
    cap_add:
      - CAP_SETUID
      - CAP_SETGID
    command:
    - rootlesskit
    - buildkitd
    - "--addr"
    - unix:///home/user/.local/share/bksock/buildkitd.sock   # <— outside XDG_RUNTIME_DIR
    - --oci-worker-no-process-sandbox
    security_opt:
    - no-new-privileges=false
    - seccomp=unconfined        # allow mount(2)
    volumes:
      # runtime dir for rootlesskit/buildkit socket
      - ./buildkit-rootless-run:/home/user/.local/run
      - /sys/fs/cgroup:/sys/fs/cgroup
      # persistent state/cache
      - ./buildkit-rootless-state:/home/user/.local/share/buildkit
      - ./buildkit-sock:/home/user/.local/share/bksock
    environment:
      XDG_RUNTIME_DIR: /home/user/.local/run
      TZ: "UTC"
      BUILDKIT_DEBUG: "1" # Optional, useful during initial testing
      BUILDKIT_EXPERIMENTAL: "1"  # if you want type=containerd exporter
    deploy:
      replicas: 1
```

## Test the Function Builder API via faas-cli

Now use faas-cli to perform a test build on the faasd host directly.

Scaffold a new function for testing:

```bash
faas-cli new --lang python3-http \
  --prefix registry:5000/functions \
  pytest
```

The `--prefix` flag is used to set prefix for the function image to our local registry.

Build the function using the function builder API and deploy it:

```bash
sudo cp /var/lib/faasd/secrets/payload-secret ./payload-secret

faas-cli up \
    --remote-builder http://127.0.0.1:8088 \
    --payload-secret ./payload-secret \
    --tag=digest
```

The `--remote-builder` flag points to the Function Builder API exposed on the local host only. This should be removed in production, and only accessed via the internal network.

The `--payload-secret` flag points to the secret you created earlier, this must be a file, not a literal string.

The `--tag=digest` flag creates a dynamic tag every time you publish a new function based upon a hash of the contents.

Test the new function:

```bash
curl -s http://127.0.0.1:8080/function/pytest
```

Then edit the message returned by the `pytest/handler.py` file, and the build again, followed by another curl.

```bash
faas-cli up \
    --remote-builder http://127.0.0.1:8088 \
    --payload-secret ./payload-secret \
    --tag=digest

curl -s http://127.0.0.1:8080/function/pytest
```

The second run will be quicker due to caching.

## Turn off access to the Function Builder API via the host

Under the `pro-builder` service in your `docker-compose.yaml` file, comment out or remove the following lines:

You should be calling the function builder via its internal service name http://pro-builder:8080

```yaml
    ports:
     - "127.0.0.1:8088:8080"
```
