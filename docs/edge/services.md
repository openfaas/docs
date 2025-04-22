# Custom Services for OpenFaaS Edge

Whilst Kubernetes uses YAML files, often templated through tools like Helm, OpenFaaS Edge uses a `docker-compose.yaml` file to configure its own core services.

Core services are the containers required to run the OpenFaaS control-plane. Additional stateful containers such as Grafana, Postgresql, Redis, etc can also be managed by OpenFaaS Edge.

The pre-installed core services include:

* OpenFaaS Gateway
* NATS
* Prometheus
* Queue Worker
* OpenFaaS Dashboard (disabled, add-on license required)
* Cron Connector (Pro version)
* faas-idler - scale to zero

Most of the commands for interacting with services, can be used with functions by passing in the `--namespace openfaas-fn` flag.

Learn more about managing services in this blog post: [How to Manage Stateful Services with OpenFaaS Edge](https://www.openfaas.com/blog/edge-services/)

## Define a service

faasd implements a sub-set of the Docker Compose spec.

The easiest way to add a new service, is to refer to the examples in the [eBook Serverless For Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else?layout=profile), or to adapt an existing service.

* `image` - full path to container image
* `command` - command to run in the container
* `environment` - environment variables to set in the container
* `volumes` - volumes to mount in the container from the host. Only files created at `/var/lib/faasd` are supported for the source path.
* `ports` - ports to expose on the host - either on all adapters or only on loopback. Most of the time, you should only ever expose ports on the loopback interface.
* `restart` - restart policy for the container - either empty to disable restarts or `always`
* `deploy.replicas` - set this value to `0` to disable a service, or remove the YAML to enable it
* `user` - user to run the container as - this must be a string, even if it is a number i.e. `"65534"`
* `gpus` - the simplest option is to provide `gpus: all` on the service i.e. ollama - [learn more about using GPUs](https://www.openfaas.com/blog/local-llm-openfaas-edge/)
* `depends_on` - a list of other services that must be running before this service starts
* `cap_add` - a list of additional Linux capabilities required for the container such as `CAP_NET_RAW`

## List services

```bash
$ faasd service list

NAME             IMAGE                                               CREATED          STATUS                 
cron-connector   ghcr.io/openfaasltd/cron-connector:0.2.9            23 seconds ago   running (23 seconds)   
faas-idler       ghcr.io/openfaasltd/faas-idler:0.5.6                23 seconds ago   running (23 seconds)   
gateway          ghcr.io/openfaasltd/gateway:0.4.38                  23 seconds ago   running (23 seconds)   
nats             docker.io/library/nats:2.10.26                      24 seconds ago   running (24 seconds)   
prometheus       docker.io/prom/prometheus:v3.2.1                    23 seconds ago   running (23 seconds)   
queue-worker     ghcr.io/openfaasltd/jetstream-queue-worker:0.3.46   23 seconds ago   running (23 seconds)   
```

## View CPU/RAM consumption for a service

```bash
$ faasd service top

NAME            PID            CPU (Cores)    Memory
prometheus      5870           2m             50 MB
gateway         5976           2m             12 MB
nats            5759           2m             4.3 MB
cron-connector  6082           0m             5.1 MB
queue-worker    6184           0m             6.3 MB
faas-idler      6493           0m             7.3 MB
```

## View logs for a service

View recent logs:

```bash
$ faasd logs NAME
```

Follow the logs:

```bash
$ faasd logs NAME --follow
```

## Restart a service

You can only use this command if a service is configured to restart automatically.

Here is how to configure the cron-connector to restart automatically.

Update the `docker-compose.yaml` file to include the `restart: always` option for the service:

```yaml
  cron-connector:
    restart: always
```

Then run:

```bash
$ faasd service restart cron-connector
```

This will kill and remove the existing container. A background task will detect the missing container for the service, and re-create it.

## Disable a service

You can disable a service, but keep it in the compose file. This is useful for testing, or for optional features.

```diff
  cron-connector:
+   deploy:
+     replicas: 0
```

Restart faasd for it to take effect.

To re-enable the service, remove those lines.

