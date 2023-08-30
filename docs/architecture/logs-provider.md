# Log Provider

You can write a custom log provider and configure the [OpenFaaS Gateway](https://github.com/openfaas/faas) to use it.

A log provider is an HTTP server that provides a `/system/logs` endpoint that supports a `GET` request with the following query parameters

* `name` : the function name and is required
* `instance` : the optional container name, that allows you to request logs from a specific function instance
* `since` : the optional datetime value to start the logs from
* `tail` : sets the maximum number of log messages to return, <=0 means unlimited
* `follow` : allows the user to request a stream of logs until the timeout

When `follow` is `true`, the server must use HTTP chunked-encoding to send a live stream of the logs.

## Configure the log provider
The `gateway` will proxy log requests to the function provider, by default. To use an alternative log provider, simply set the `logs_provider_url` environment variable in your `gateway` server.  The `gateway` will then proxy the logs requests to this URL.

## Available providers

* [faas-netes](https://github.com/openfaas/faas-netes) is the Kubernetes provider and queries logs from the Kubernetes API

* [faasd](https://github.com/openfaas/faasd) queries the logs from the journal, stored by functions and core services

* [openfaas-loki](https://github.com/LucasRoesler/openfaas-loki) is a community provider which uses [Grafana Loki](https://github.com/grafana/loki) to collection and query the function logs

## Create a new log provider

The [`github.com/openfaas/faas-provider/logs`](https://github.com/openfaas/faas-provider/tree/master/logs) package provides a Go interface and utilities to simplify the creation of a new log provider.  Once you have implemented the `Requester` interface the other package utilities can be used to create the required http server. A very simple ["static" logs example can be found in the `faas-provider` repo](https://github.com/openfaas/faas-provider/tree/master/logs/example).
