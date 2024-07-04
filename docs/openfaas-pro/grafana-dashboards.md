# Grafana dashboards

OpenFaaS comes with [built-in Prometheus metrics](/architecture/metrics). We provide a collection of Grafana dashboards to customers to help with monitoring OpenFaaS and individual functions.

Available dashboards:

- Autoscaling and usage metrics dashboard - Get an overview of all deployed functions and metrics for the OpenFaaS gateway.
- Spotlight dashboard - Get detailed CPU/RAM and Rate, Error, Duration (RED) metrics for individual functions. 
- Queue worker dashboard - Monitor the work queue for your async function invocations.
- Builder dashboard - Dashboard for monitoring the OpenFaaS [Function Builder API](/openfaas-pro/builder).

See also:

* [Grafana Dashboard JSON files in the Customer Community](https://github.com/openfaas/customers/tree/master/dashboards)

## Autoscaling and usage metrics dashboard

The usage metrics dashboard gives you an overview of all the function that you have deployed. It gives you insights into:

- CPU/RAM usage of functions.
- The function replica count and current load.
- Rate, Error, Duration (RED) metrics for the gateway API and individual functions.

![Autoscaling and usage metrics dashboard](/images/grafana/overview-dashboard.png)
> Autoscaling and usage metrics dashboard

## Spotlight dashboard

The spotlight dashboard lets you filter out the data for a single function that you select from a dropdown. This allows you to zoom in and get a good view of what's going on with one function in isolation.

![Spotlight dashboard](/images/grafana/spotlight-dashboard.png)
> Spotlight dashboard

## Queue-worker dashboard

The JetStream queue-worker comes with the addition of metrics to monitor the behaviour of queues. This dashboard can help you get insights in things like:

- The message ingestion rate for each queue.
- The number of messages waiting in the queue to be processed by functions. 
- The processing rate.

![Queue-worker dashboard](/images/grafana/jetstream-queue-worker-dashboard.png)
> Queue-worker dashboard

## Builder dashboard

Dashboard for the [OpenFaaS Pro function builder](/openfaas-pro/builder). The builder dashboard can give you better insights in the performance of your builder pods. These metrics include:

- The average build duration per pod
- The invocation rate
- The number of builds in progress

![Builder dashboard](/images/grafana/builder-dashboard.png)
> Builder dashboard