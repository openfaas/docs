# Grafana dashboards

OpenFaaS comes with [built-in Prometheus metrics](/architecture/metrics). We provide a collection of Grafana dashboards to customers to help with monitoring OpenFaaS and individual functions.

Available dashboards:

- Autoscaling and usage metrics dashboard - Get an overview of all deployed functions and metrics for the OpenFaaS gateway.
- Spotlight dashboard - Get detailed CPU/RAM and Rate, Error, Duration (RED) metrics for individual functions. 
- Queue worker dashboard - Monitor the work queue for your async function invocations.
- Builder dashboard - Dashboard for monitoring the OpenFaaS [Function Builder API](/openfaas-pro/builder).

## Autoscaling and usage metrics dashboard

The usage metrics dashboard gives you an overview of all the function that you have deployed. It gives you insights into:

- CPU/RAM usage of functions.
- The function replica count and current load.
- Rate, Error, Duration (RED) metrics for the gateway API and individual functions.

![Autoscaling and usage metrics dashboard](https://camo.githubusercontent.com/e45062a432a6363a806d07bac9e9d61a46b7dc0ac0cd6d9496cff16704a46186/68747470733a2f2f7777772e6f70656e666161732e636f6d2f696d616765732f323032322d30352d6261636b70726573737572652f64617368626f6172642e706e67)
> Autoscaling and usage metrics dashboard

## Spotlight dashboard

The spotlight dashboard lets you filter out the data for a single function that you select from a dropdown. This allows you to zoom in and get a good view of what's going on with one function in isolation.

![Spotlight dashboard](https://user-images.githubusercontent.com/6358735/171370016-fc1e305c-d57f-4cb4-8cb6-2d85379985bd.png)
> Spotlight dashboard

## Queue-worker dashboard

The JetStream queue-worker comes with the addition of metrics to monitor the behaviour of queues. This dashboard can help you get insights in things like:

- The message ingestion rate for each queue.
- The number of messages waiting in the queue to be processed by functions. 
- The processing rate.

![Queue-worker dashboard](https://camo.githubusercontent.com/fc802102ec26e9a7ca8fd3499e92852ab0f50340b99a54c8bdaae581543c2f8e/68747470733a2f2f7777772e6f70656e666161732e636f6d2f696d616765732f323032322d30372d6a657473747265616d2d666f722d6f70656e666161732f71756575652d776f726b65722d64617368626f6172642e706e67)
> Queue-worker dashboard

## Builder dashboard

Dashboard for the [OpenFaaS Pro function builder](/openfaas-pro/builder). The builder dashboard can give you better insights in the performance of your builder pods. These metrics include:

- The average build duration per pod
- The invocation rate
- The number of builds in progress

![Builder dashboard](https://user-images.githubusercontent.com/16267532/173072087-bb2bc408-245c-4934-a828-2e1580fbc3db.png)
> Builder dashboard