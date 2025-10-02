# OpenFaaS Edge Overview

[OpenFaaS Edge](/deployment/edge) is a commercial distribution of both [OpenFaaS Standard](/docs/openfaas-pro/) and [faasd](https://github.com/openfaas/faasd) designed for redistribution of bespoke solutions to end customers.

The handbook and reference documentation for OpenFaaS Edge are available in the [Serverless for Everyone Else eBook](https://store.openfaas.com/l/serverless-for-everyone-else?layout=profile). As a customer, you will receive a 100% discount code for the eBook.

Most of the [OpenFaaS Pro documentation](/docs/openfaas-pro/) and [Helm charts](https://github.com/openfaas/faas-netes/tree/master/chart) can be used or adapted, however you'll find some specifics here:

## Packaging/deployment

* [OpenFaaS Edge Deployment](/deployment/edge)
* [Upgrading OpenFaaS Edge - itself and its containers](/deployment/edge/#upgrading-openfaas-edge)
* [Preloading functions for distribution](/edge/preloading)
* [Air Gap](/edge/airgap)
* [Custom DNS servers](/edge/custom-dns)
* [Improve container security with gVisor](/edge/gvisor)
* [Troubleshooting an installation](/edge/troubleshooting)

## Build Functions From Source

* [Function Builder API for OpenFaaS Edge](/edge/builder) - build functions as non-root, from source code using a variety of templates

## OpenFaaS Edge guides 

* [Services](/edge/services) - stateful containers defined in `docker-compose.yaml`
* [TLS](/edge/tls) - secure the gateway for requests from the Internet
* [Scale to Zero for OpenFaaS Edge](/edge/scale-to-zero)
* [Kafka Connector for OpenFaaS Edge](/edge/kafka-deployment)
* [GPU support for services](/edge/gpus) - package i.e. Ollama and an LLM for use by functions
* [OpenTelemetry](/edge/open-telemetry)
* [Resource constraints](/reference/yaml/#function-memorycpu-limits) - limit RAM/CPU consumption by functions

## Looking for something else?

Reach out to us via contact@openfaas.com with comments, questions or suggestions for additional content.
