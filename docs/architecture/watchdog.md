## OpenFaaS watchdog

The OpenFaaS watchdog is responsible for starting and monitoring functions in OpenFaaS. Any binary can become a function through the use of watchdog.

The watchdog becomes an "init process" with an embedded HTTP server written in Golang, it can support concurrent requests, timeouts and healthchecks. The newer of-watchdog mentioned below is similar, but ideal for streaming use-cases or when an expensive resource needs to be maintained between requests such as a database connection, ML model or other data. 

## Classic watchdog

The classic watchdog is currently used in all the official OpenFaaS templates. You can view the templates in the [templates repository](https://github.com/openfaas/templates).

<a href="https://camo.githubusercontent.com/61c169ab5cd01346bc3dc7a11edc1d218f0be3b4/68747470733a2f2f7062732e7477696d672e636f6d2f6d656469612f4447536344626c554941416f34482d2e6a70673a6c61726765"><img src="https://camo.githubusercontent.com/61c169ab5cd01346bc3dc7a11edc1d218f0be3b4/68747470733a2f2f7062732e7477696d672e636f6d2f6d656469612f4447536344626c554941416f34482d2e6a70673a6c61726765"></a>

*Pictured: technical conceptual diagram of the OpenFaaS watchdog during an invocation*

Technical documentation on the classic watchdog is available along with a table with all configuration options.

* [Watchdog README](https://github.com/openfaas/faas/tree/master/watchdog)

## of-watchdog

of-watchdog is the next-generation of the OpenFaaS watchdog component hosted in the [openfaas-incubator organisation](https://github.com/openfaas-incubator).

<a href="/architecture/watchdog-modes.png"><img src="/architecture/watchdog-modes.png"></a>

*Pictured: various modes for the of-watchdog component*

This version of the of-watchdog brings new features for high-throughput and enables re-use of expensive resources such as database connection pools or machine-learning models. The primary difference is the ability to keep the function process warm between invocations. The classic watchdog forks one process per request giving the highest level of portability, but the newer version enables a `http` mode where that same process can be re-used repeatedly to offset the latency of forking.

* [Read more on the of-watchdog](https://github.com/openfaas-incubator/of-watchdog)

The `http` mode of the of-watchdog uses [custom templates](https://github.com/openfaas-incubator/of-watchdog#2-http-modehttp).
