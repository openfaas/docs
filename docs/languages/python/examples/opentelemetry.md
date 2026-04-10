Using [OpenTelemetry zero-code instrumentation](https://opentelemetry.io/docs/zero-code/python/) for python functions requires some minor modifications to the existing Python templates.

There are two ways to [customise a template](/cli/templates/#how-to-customise-a-template):

- Fork the template repository and modify the template. Recommended method that allows for distribution and reuse of the template.
- Pull the template and apply patches directly in the `./template/<language_name>` directory. Good for quick iteration and experimentation with template modifications. The modified template can not be shared and reused. Changes may get overwritten when pulling templates again.

Add the required packages for auto instrumentation to the `requirements.txt` file of the template:

```
opentelemetry-distro
opentelemetry-exporter-otlp
```

Update the Dockerfile to run the bootstrap command after the the template and function packages have been installed:

```diff
# Build the function directory and install any user-specified components
USER app

RUN mkdir -p function
RUN touch ./function/__init__.py
WORKDIR /home/app/function/
COPY --chown=app:app function/requirements.txt	.
RUN pip install --no-cache-dir --user -r requirements.txt
+ RUN opentelemetry-bootstrap -a install
```

The `opentelemetry-bootstrap -a install` command reads through the list of packages installed in your active site-packages folder, and installs the corresponding instrumentation libraries for these packages, if applicable. The OpenTelemetry Python agent uses [monkey patching](https://stackoverflow.com/questions/5626193/what-is-monkey-patching) to modify functions in these libraries at runtime.

Update the fprocess ENV in the Dockerfile to start the OpenTelemetry agent:

```diff
# configure WSGI server and healthcheck
USER app

- ENV fprocess="python index.py"
+ ENV fprocess="opentelemetry-instrument python index.py"
```

Use your modified template to create a new function.

The OpenTelemetry agent can be configured using environment variables on the function:

```diff
functions:
  greet:
    lang: python3-http-otel
    handler: ./greet
    image: greet:latest
+    environment:
+      OTEL_SERVICE_NAME: greet.${NAMESPACE:-openfaas-fn}
+      OTEL_TRACES_EXPORTER: console,otlp
+      OTEL_METRICS_EXPORTER: none
+      OTEL_LOGS_EXPORTER: none
+      OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_EXPORTER_OTLP_ENDPOINT:-collector:4317}
```

- `OTEL_SERVICE_NAME` sets the name of the service associated with the telemetry and is used to identify telemetry for a specific function. It can be set to any value you want be we recommend using the clear function identifier `<fn-name>.<fn-namespace>`.
- `OTEL_TRACES_EXPORTER` specifies which tracer exporter to use. In this example traces are exported to `console` (stdout) and with `otlp`. The `otlp` option tells `opentelemetry-instrument` to send the traces to an endpoint that accepts OTLP via gRPC.
- setting `OTEL_METRICS_EXPORTER` and `OTEL_LOGS_EXPORTER` to `none` we disable the metrics and logs exporters. You can enable them if desired.
- `OTEL_EXPORTER_OTLP_ENDPOINT` sets the endpoint where telemetry is exported to.

To see the full range of configuration options, see [Agent Configuration](https://opentelemetry.io/docs/zero-code/python/configuration/)
