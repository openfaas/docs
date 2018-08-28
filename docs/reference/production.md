# Moving to production

Recommendations for moving to a production environment.

## Pick an orchestrator

Pick either Docker Swarm or Kubernetes. Some of our users prefer Docker Swarm, but Kubernetes provides the most complete experience.

## Configure the gateway

### High availability (HA)

Scale your gateway to at least 2 replicas so that you have high-availability. If one container or Node crashes/disconnects then you have a backup ready.

### Timeouts

Set an appropriate timeout in the gateway configuration for the maximum time needed for your functions. This can be reconfigured later with a rolling-update.

### Authentication

You must enable authentication on the gateway to prevent unauthorized users from deploying, replacing or discovering your functions.

### SSL

Enable SSL on your gateway to make sure that data in-bound is encrypted.

## Enable NetworkPolicy (Kubernetes)

Kubernetes allows policies to be built at the network level which can prevent functions talking back to the OpenFaaS Core services and if desired prevent functions from invoking other functions. This is recommended to reduce privileges. 

## Monitor your functions

Install the Grafana dashboard and monitor your functions for non-200 error codes and for excessive durations or timeouts.

## Configure the queue-worker / NATS Streaming

If you wish to use asynchronous invocation using NATS Streaming then you should attach a persistent volume to the container so that any unfinished tasks are still available after the container is rescheduled or restarted.
