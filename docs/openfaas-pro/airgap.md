## Air Gap with Kubernetes

OpenFaaS Standard and OpenFaaS for Enterprises are licensed for use within an airgap when purchased on an annual basis via invoice.

You can use your own choice of tooling to mirror images and bundle the Helm chart(s) required for your installations, or you can use our own purpose-built airfaas tool.

### Mirror OpenFaaS images into your own registry

If your organisation has a policy of mirroring all consumed images from vendors into a private registry, then `airfaas mirror` can help you with this, even if you don't install OpenFaaS into an airgapped environment.

```bash
faas-cli plugin get airfaas
```

Mirror all images for the given chart/index to a custom registry:

```bash
airfaas download images \
        openfaas/openfaas \
        --registry-prefix ttl.sh
```

Mirror the Kafka-connector's images:

```bash
airfaas download images \
        openfaas/kafka-connector \
        --registry-prefix ttl.sh
```

The `--url` flag can be used to specify a different Helm chart repository. The only requirement is that images are stored in the same format as OpenFaaS: i.e. `image:` or `componentName.image:`.

After the mirroring is complete, you'll receive output in the format of a values.yaml file, which you can add to your `helm upgrade --install` command.

```bash
$ faas-cli airfaas mirror openfaas/openfaas --to https://aws_account_id.dkr.ecr.us-west-2.amazonaws.com/openfaas
Mirrored 18 images in 3m7.242s
```

Then copy the below to i.e. `values-mirror.yaml`:

```yaml
gatewayPro:
  image: aws_account_id.dkr.ecr.us-west-2.amazonaws.com/openfaas/openfaasltd/gateway:0.4.27
alertmanager:
  image: aws_account_id.dkr.ecr.us-west-2.amazonaws.com/openfaas/prom/alertmanager:v0.27.0
autoscaler:
  image: aws_account_id.dkr.ecr.us-west-2.amazonaws.com/openfaas/openfaasltd/autoscaler:0.3.6
```

Your final OpenFaaS Pro installation may look something like, where you use the standard `values-pro.yaml` file, your own settings in `values-staging.yaml`, then finally the mirrored images overlaid over that in `values-mirror.yaml`:

```bash
helm upgrade --install openfaas \
  --install openfaas/openfaas \
  --namespace openfaas \
  -f values-pro.yaml \
  -f values-staging.yaml \
  -f values-mirror.yaml
```

### Perform an offline installation

Airfaas can perform an offline installation of OpenFaaS into an airgapped environment. It will bundle the Helm chart(s) and images required for the installation, then restore the images into an offline registry, and install OpenFaaS using the Helm chart(s) from the local filesystem.

![Conceptual download process](https://www.openfaas.com/images/2024-04-airgap/download.png)
> The download process

![Conceptual upload process](https://www.openfaas.com/images/2024-04-airgap/restore.png)
> The upload / installation process

Follow the instructions in: [Deploy airgapped Serverless Functions with OpenFaaS](https://www.openfaas.com/blog/airgap-serverless-functions/)