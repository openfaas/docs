# Long term OpenFaaS metrics retention

OpenFaaS exposes metrics in Prometheus format from its different components. These metrics are collected by a built-in Prometheus server which is deployed via the OpenFaaS Helm chart.

The built-in Prometheus instance is designed to be a "black box", an internal component of OpenFaaS, not to be used for long term storage or custom settings. By default there is no persistence for the Prometheus metrics and the metrics retention period is only 15 days.

But what if you would like to enable long-term retention of Prometheus metrics?

In this tutorial we will show you how metrics can be exported using [Grafana Alloy](https://grafana.com/docs/alloy/latest/) or [Prometheus Federation](https://prometheus.io/docs/prometheus/latest/federation/) for long term retention.

## Configure retention for OpenFaaS built-in Prometheus

You can configure an optional PVC for Prometheus to persist data for the OpenFaaS Prometheus deployment. 

Without the PVC, the data held by Prometheus is erased if:

- The node where Prometheus is running gets reclaimed or recycled
- The Helm chart is upgraded (something we should be doing often for fixes and features)
- You update the configuration for Prometheus for custom autoscaling rules etc

You can configure this feature through the [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas). By default it is turned off, but we recommend turning it on.

```yaml
prometheus:
  # Set to true to enable persistent storage for the Prometheus Pod
  # otherwise, the data will be lost when the Pod is restarted
  pvc:
    enabled: false
    # You may want to set this higher for production, or lower for development/staging.
    size: 30Gi
    # Leave the storageClassName blank for the default storage class
    # using the string "default" does not necessarily mean the default
    # storage class
    storageClassName:
```

The retention period can also be changed;:

```yaml
prometheus:
  retention:
    time: 15d
```

## Prometheus metrics federation

Federation allows Prometheus to scrape selected time series from another Prometheus server. You can configure your own Prometheus deployment to scrape metrics from the OpenFaaS Prometheus server.

Add the following job to your Prometheus scrape configuration: 

```yaml
scrape_configs:
  - job_name: 'openfaas'
    scrape_interval: 60s

    honor_labels: true
    metrics_path: "/federate"

    params:
      'match[]':
        - '{job=~\".*\"}'
        - '{__name__=~\"^job:.*\"}'
    
    static_configs:
      - targets:
        - 'prometheus.openfaas.svc.cluster.local:9090'
```

This config assumes your Prometheus deployment is running in the same cluster as OpenFaaS. If this is not the case we recommend using [Grafana Alloy](https://grafana.com/docs/alloy/latest/) to collect metrics and forward them to a Prometheus-compatible database.

## Collect Prometheus metrics with Grafana Alloy

We will be deploying and configuring Grafana Alloy through Helm and configure it to collect metrics from the OpenFaaS Prometheus server. Those metrics are then forwarded to a remote Prometheus server using the [Prometheus Remote Write protocol](https://prometheus.io/docs/specs/remote_write_spec/).

We will show you how to configure different remote write targets like [Grafana Cloud](https://grafana.com/products/cloud/) and [Amazon Managed Service for Prometheus](https://aws.amazon.com/prometheus/).

### Deploy Grafana Alloy with Helm

Create a `values.yaml` file to deploy and configure Grafana Alloy for remote write to an external Prometheus deployment with the Grafana Alloy Helm chart.

We configure Alloy directly using the Helm configuration but you can create a separate ConfigMap from a file if that is preferred. See [Configure Grafana Alloy on Kubernetes](https://grafana.com/docs/alloy/latest/configure/kubernetes/) for more info.

```sh
export PROM_USERNAME=""
export PROM_PASSWORD="./prom-password.txt"
export REMOTE_WRITE_ENDPOINT=""

cat <<EOF > values.yaml
alloy:
  configMap:
    content: |-
      // Collect metrics from the OpenFaaS Prometheus instance and forward them
      // to the prometheus remote write component for long term retention.
      prometheus.scrape "openfaas" {
        targets    = [{
          __address__ = "prometheus.openfaas.svc.cluster.local:9090",
        }]
        honor_labels = true
        metrics_path = "/federate"

        params = {
          "match[]" = ["{job=~\".*\"}", "{__name__=~\"^job:.*\"}"],
        }

        scrape_interval = "60s"

        forward_to = [prometheus.remote_write.metrics_service.receiver]  
      }

      prometheus.remote_write "metrics_service" {
        endpoint {
          url = ${REMOTE_WRITE_ENDPOINT}

          basic_auth {
            username = ${PROM_USERNAME}
            password = $(cat $PROM_PASSWORD)
          }
        }
      }
EOF
```

You can leave out the `basic_auth` section if your Prometheus deployment has no authentication enabled.

!!! Note
    Check out the Alloy [values.yaml](https://raw.githubusercontent.com/grafana/alloy/main/operations/helm/charts/alloy/values.yaml) for more configuration options.

The target Prometheus server should be running with the flag `--web.enable-remote-write-receiver=true` to accept remote write requests.

Deploy Grafana Alloy in the `openfaas` namespace:

```sh
helm upgrade grafana-alloy \
  --install grafana/alloy \
  --namespace openfaas \
  -f ./values.yaml
```

Optionally verify the configuration through the Alloy UI.

```sh
kubectl port-forward -n openfaas svc/grafana-alloy 12345:12345
```

### Grafana Cloud

The configuration for pushing metrics to Grafana Cloud is exactly the same as for a self-hosted Prometheus instance. You will need to get the remote write endpoint, username (instance ID) and password (API access token) from your Grafana Cloud organization.

### Amazon Managed Service for Prometheus (from EKS cluster or EC2)

It is possible to ingest OpenFaaS Prometheus metrics in [Amazon Managed Service for Prometheus](https://aws.amazon.com/prometheus/) for clusters running on Amazon EKS or self-managed cluster running on Amazon EC2.

1. Create a new Amazon Managed Service for Prometheus workspace in your desired region.
2. [Set up IAM roles for service accounts](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-existing-Prometheus.html#AMP-onboard-existing-Prometheus-IRSA)

Make sure you have created an IAM role `amp-iamproxy-ingest-role` before continuing.

This IAM role must be associated with the Kubernetes service account used by Grafana Alloy to give it the appropriate permissions to push metrics to the AWS Prometheus service.

Append the parameters for the service account to your Helm configuration file for Grafana Alloy:

- Replace `${IAM_PROXY_PROMETHEUS_ROLE_ARN}` with the ARN of the `amp-iamproxy-ingest-role` that you created.

```yaml
serviceAccount:
    annotations:
        eks.amazonaws.com/role-arn: ${IAM_PROXY_PROMETHEUS_ROLE_ARN}
```

Add a new remote write target to the Alloy config:

- Replace `${WORKSPACE_ID}` with your Amazon Managed Service for Prometheus workspace ID.
- Replace `${REGION}` with the Region of the Amazon Managed Service for Prometheus workspace (such as us-west-2).

```
prometheus.remote_write "aws" {
    endpoint {
        url = "https://aps-workspaces.${REGION}.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write"
    }

    // Configure AWS Signature Verification 4 for authenticating to the endpoint.
    sigv4 {
        region = "${AWS_REGION}"
    }

    // Configuration for how metrics are batched before sending.
    queue_config {
        max_samples_per_send = 1000
        max_shards = 200
        capacity = 2500
    }
}
```

Change the `prometheus.scrape "openfaas"` component to forward metrics to the correct target:

```diff
prometheus.scrape "openfaas" {
-  forward_to = [prometheus.remote_write.metrics_service.receiver]
+  forward_to = [prometheus.remote_write.aws.receiver]
}
```

Create or update the Grafana Alloy deployment:

```sh
helm upgrade grafana-alloy \
  --install grafana/alloy \
  --namespace openfaas \
  -f ./values.yaml
```
