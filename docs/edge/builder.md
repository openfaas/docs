# Function Builder API for OpenFaaS Edge

The Function Builder API is available in OpenFaaS Edge and allows you to build functions from source code using a variety of templates.

The builder runs as a non-root user making use of user namespaces in Linux.

## Prerequisites

* An OpenFaaS for Enterprises license or an additional entitlement for the Function Builder API is required to use this feature.
* Your operating system must support user namespaces, generally most modern Linux distributions do.
* Docker must not be installed on the host system.
* faasd-pro version 0.2.23 or later is required.

## Create a registry secret

For testing purposes, you can use an ephemeral registry which requires no authentication such as [ttl.sh](https://ttl.sh).

Bear in mind that this ephemeral cluster is public, and have much more latency than your final production setup.

```bash
sudo tee /var/lib/faasd/secrets/docker-config <<EOF
{
}
```

For production use, create a secret with a proper authenticated registry, see the notes on the [Function Builder API for Kubernetes](/openfaas-pro/builder).

## Create a payload secret

The payload secret will be used to sign the payloads sent to the Function Builder's API.

```bash
openssl rand -base64 32 | sudo tee /var/lib/faasd/secrets/payload-secret
```

## Update your docker-compose.yaml

Add the following services to your `docker-compose.yaml` file:

```yaml
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

```bash
faas-cli new --lang python3-http \
  --prefix ttl.sh/openfaas-tests \
  pytest

sudo cp /var/lib/faasd/secrets/payload-secret ./payload-secret

faas-cli up \
    --remote-builder http://127.0.0.1:8088 \
    --payload-secret ./payload-secret
```

## Turn off access to the Function Builder API via the host

Under the `pro-builder` service in your `docker-compose.yaml` file, comment out or remove the following lines:

You should be calling the function builder via its internal service name http://pro-builder:8080

```yaml
    ports:
     - "127.0.0.1:8088:8080"
```

