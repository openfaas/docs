# Collect OpenTelemetry data from functions

To collect OpenTelemetry data for functions and services on OpenFaaS Edge you can deploy an OpenTelemetry [Collector](https://opentelemetry.io/docs/collector/) as additional service.

## Deploy Grafana Alloy

We are going to show the configuration for [Grafana Alloy](https://grafana.com/oss/alloy-opentelemetry-collector/) it is possible to different collectors as well.

Add the following to the services section in the `docker-compose.yaml`:

```yaml
services:
  alloy:
    image: docker.io/grafana/alloy:v1.8.2
    user: "65534"
    command:
      - "alloy"
      - "run"
      - "/etc/alloy/config.alloy"
      - "--storage.path=/tmp/alloy"
      - "--server.http.listen-addr=0.0.0.0:12345"
      - "--server.http.ui-path-prefix=/"
      - "--stability.level=generally-available"
    volumes:
      - type: bind
        source: ./config.alloy
        target: /etc/alloy/config.alloy
    restart: always
    ports:
      - "127.0.0.1:12345:12345"
```

The `ports` section is optional and makes the Alloy UI, which is served on port `12345` accessible on the host.

The Alloy configuration is bind mounted into the service. Create the Alloy configuration file at `/var/lib/faasd/config.alloy`':

``` 
otelcol.exporter.otlp "grafanacloud" {
  client {
    endpoint = "<endpoint>"
    auth = otelcol.auth.basic.grafanacloud.handler
  }
}

otelcol.auth.basic "grafanacloud" {
  username = "<username>"
  password = "<api-token>"
}

otelcol.receiver.otlp "otlp_receiver" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}
```

This is an example of a basic Alloy configuration that exports traces to [Grafana Cloud](https://grafana.com/products/cloud/). You will need to modify the config to with your own Grafana Cloud Tempo service endpoint and credentials or configure an exporter for a different local or cloud backend.

- The [otelcol.exporter.otlp](https://grafana.com/docs/alloy/latest/reference/components/otelcol/otelcol.exporter.otlp/) component configures an OpenTelemetry Protocol (OTLP) exporter to send data to a backend for storage.

- Production-ready Alloy configurations shouldn’t send OpenTelemetry data directly to an exporter for delivery. Instead, data is usually sent to one or more processor components that perform various transformations on the data.

    [otelcol.processor.batch](https://grafana.com/docs/alloy/latest/reference/components/otelcol/otelcol.processor.batch/) is used to batch data before sending it to the exporter to reduce the number of network requests.

- [otelcol.receiver.otlp](https://grafana.com/docs/alloy/latest/reference/components/otelcol/otelcol.receiver.otlp/) is used to configure Alloy to receive telemetry data over the network using the OpenTelemetry Protocol. In this case we allow applications to send data over either HTTP or gRPC. Both endpoints listen for traffic on all interfaces.

Checkout the docs for more details on [how to configure Grafana Alloy to collect OpenTelemetry data](https://grafana.com/docs/alloy/latest/collect/opentelemetry-data/)

Functions and services deployed on OpenFaaS Edge can use the service name when configuring the telemetry collection endpoint e,g. `alloy:4317` or `alloy:4318`. 

For more info on how to instrument and configure telemetry for functions checkout our language guides for: [Python](/languages/python/#opentelemetry-zero-code-instrumentation) or [Node.js](/languages/node/#opentelemetry-zero-code-instrumentation).