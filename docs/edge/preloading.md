# Preloading Functions for Distribution

OpenFaaS Edge is licensed for distribution of bespoke solutions to end customers. The product itself can shipped as a convenient DEB/RPM package, or pre-installed in a VM image.

But when it comes to functions, you generally have to deploy these via `faas-cli`.

Preloading is a solution to this problem.

On the first boot of the `faasd-provider` service, any YAML files found will be deployed automatically.

So before you install OpenFaaS Edge, create the: `/var/lib/faasd-provider/functions` folder, and place any number of OpenFaaS function YAML files within it.

## A single stack.yaml

You can ship a single stack.yaml file with all your functions defined separately within it.

*stack.yaml*

```yaml
functions:
  env:
    image: ghcr.io/openfaas/env:latest
    fprocess: env
  nodeinfo:
    image: ghcr.io/openfaas/nodeinfo:latest
```

You can leave in build data such as the template and handler folder, or you can trim it away for brevity. It won't be used at deployment time.

Copy the files to the destination:

```bash
sudo mkdir -p /var/lib/faasd-provider/functions/
sudo cp stack.yaml /var/lib/faasd-provider/functions/
```

## Multiple YAML files

You could also create multiple YAML files - one per function:

*env.yaml*

```yaml
functions:
  env:
    image: ghcr.io/openfaas/env:latest
    fprocess: env
```

*nodeinfo.yaml*

```yaml
functions:
  nodeinfo:
    image: ghcr.io/openfaas/nodeinfo:latest
    fprocess: nodeinfo
```

Copy the files to the destination:

```bash
sudo mkdir -p /var/lib/faasd-provider/functions/
sudo cp nodeinfo.yaml /var/lib/faasd-provider/functions/
sudo cp env.yaml /var/lib/faasd-provider/functions/
```

## Running the preload a second time

By default, the preload will only run once, then a file is written out to prevent it from running again.

To have the preload run a second time i.e. during testing, simply remove the run file: `sudo rm -rf /var/lib/faasd-provider/bootstrap.ran`.

