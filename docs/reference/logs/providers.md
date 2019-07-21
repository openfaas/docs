# Log Providers

OpenFaaS Gateway [0.15.0+](https://github.com/openfaas/faas/releases/tag/0.15.0) supports pluggable logging providers to support streaming function logs.

A log provider is an HTTP server that provides a `/system/logs` endpoint that supports a `GET` request with the following query parameters

* `name` : the function name and is required
* `instance` : the optional container name, that allows you to request logs from a specific function instance
* `since` : the optional datetime value to start the logs from
* `tail` : sets the maximum number of log messages to return, <=0 means unlimited
* `follow` : allows the user to request a stream of logs until the timeout

When `follow` is `true`, the server must use HTTP chunked-encoding to send a live stream of the logs.

## Configuring the log provider
The `gateway` will proxy log requests to the function provider, by default. To use an alternative log provider, simply set the `logs_provider_url` environment variable in your `gateway` server.  The `gateway` will then proxy the logs requests to this URL.

## Available providers

* `faas-netes` : the official Kubernetes function provider also provides logs by directly querying the Kubernetes cluster API
* `faas-swarm` : the official Swarm function provider also provides logs by directly querying the Swarm cluster API
* [`openfaas-loki`](https://github.com/LucasRoesler/openfaas-loki) : a community developed provider, uses [Grafana Loki](https://github.com/grafana/loki) to collection and query the function logs

## Creating a new provider

The [`github.com/openfaas/faas-provider/logs`](https://github.com/openfaas/faas-provider/tree/master/logs) package provides a Go interface and utilities to simplify the creation of a new log provider.  Once you hae implemented the `Requester` interface

```sh
type Requester interface {
	// Query submits a log request to the actual logging system.
	Query(context.Context, Request) (<-chan Message, error)
}
```

the other package utilities can be used to create the required http server. A very simple ["static" logs example can be found in the `faas-provider` repo](https://github.com/openfaas/faas-provider/tree/master/logs/example).
