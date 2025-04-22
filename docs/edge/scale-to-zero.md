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

