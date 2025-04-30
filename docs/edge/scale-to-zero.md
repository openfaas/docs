# Scale to Zero for OpenFaaS Edge

OpenFaaS Edge ships a simplified version of scale to zero, which supports a global idle value for all functions.

OpenFaaS on Kubernetes supports fine-grained idle times for each function.

To enable scale to zero for a function, add the following label:

* `com.openfaas.scale.zero=true`

This can be set via the CLI using flags, REST API, or via stack.yaml.

The global idle period can be adjusted by editing the `faas-idler` section of `/var/lib/faasd/docker-compose.yaml`

```yaml
      # If a function is inactive for x minutes, it may be scaled to zero
      - "inactivity_duration=10m"
      # The interval between each attempt to scale functions to zero
      - "reconcile_interval=5m" 
      # Write additional debug information
      - "write_debug=false"
```

## Scaling up from zero

Functions are automatically scaled up from zero when there is a request. By default the OpenFaaS gateway probes the function ready endpoint to know if a function is ready to start accepting requests before forwarding the request.

The watchdog implements the default ready endpoint `/_/ready`. You can override the endpoint by setting the annotation `com.openfaas.ready.http.path` on the function. A custom ready endpoint can be used if you have a function that is slow to start. 

!!! note "Scale from zero with gVisor"

    Some languages, like NodeJS, take longer to initialize when using the gVisor runsc runtime. The watchdog reports ready while the function process is still initializing. It is recommended to implement and configure a custom readiness endpoint on the function. 

To extend the probing duration adjust the global probing configuration by editing the `gateway` section of `/var/lib/faasd/docker-compose.yaml`

```yaml
services:
  gateway:
    environment:
      # The interval between probes
      - probe_interval=100ms
      # Max number of probes
      - probe_count=20

```

To turn of function probing set `probe_functions=false`.

