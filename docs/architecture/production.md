# Deploy OpenFaaS in Production

This page contains recommendations for deploying OpenFaaS for production usage. It is by no means exhaustive and comes with no warranty or liability.

## Decide on your Kubernetes flavour

When deploying OpenFaaS it is recommended that you use [Kubernetes](https://kubernetes.io/). A managed Kubernetes service such as GKE, EKS is recommended if you are using public cloud. You can also deploy to your own infrastructure.

These instructions apply for both Kubernetes and OpenShift 3.x.

> Note: Docker Swarm and other providers are available, but are beyond the scope of this document.

### Chart options

The `helm` chart is the most flexible way to deploy OpenFaaS and allows you to customize many different options such as whether scale-to-zero is enabled, how many replicas to have of each component and what kind of healthchecks or probes to use.

Depending on your existing usage of `helm` or your preferences you may not want to use the *Tiller* component of helm. In this case you can use `helm template` in the deployment guide to generate YAML files to apply on your cluster.

You will find each helm chart option documented in the OpenFaaS chart:

> See also: [OpenFaaS Chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#configuration)

#### High-availability (HA)

It is recommended that you have high-availability for the OpenFaaS components including:

* OpenFaaS gateway

    The gateway supports more than one replica and can be auto-scaled with HPAv2

* OpenFaaS queue-worker

    Scale the queue worker replicas according to the maximum level of asynchronous tasks you want to process in parallel. 

You may also want to set a minimum availability level for each function of > 1.

See the chart options for how to set the number of replicas.

#### Configure timeouts

Configure each timeout as appropriate for your workloads. If you expect all functions or endpoints to return within a few seconds then set a timeout to accommodate that. If you expect all workloads to run for several minutes then bear that in mind with your values.

> Note: You may have a timeout configured in your IngressController or cloud LoadBalancer. See your documentation or contact your administrator to check the value and make sure they are in-sync.

#### Configure function health-check probes

There are two types of health-check probes: exec and http. The default in the helm chart is exec for backwards-compatibility.

It is recommended to use httpProbes by default which are more CPU efficient than exec probes. If you are using [Istio](https://istio.io) with mutual TLS then you will need to use exec probes.

#### Configure function health-check interval

There are both liveness and readiness checks for functions. If you are using the scale-from-zero feature then you may want to set these values as low as possible, however this may require additional CPU. A sensible default would be something that makes sense for your own use-case. Tune each value as needed.

#### Non-root user for functions

Set the non-root user flag so that every function is forced to run in a restricted security context.

#### Choose the operator or faas-netes

The faas-netes controller for OpenFaaS is the most mature, well tested and supported. You may also use the OpenFaaS Operator if you would like to use a Function CRD. If you are not sure which to use, then use faas-netes.

> See also: [faas-netes](https://github.com/openfaas/faas-netes), [openfaas-operator](https://github.com/openfaas-incubator/openfaas-operator).

#### Single or multi-stage?

Now decide if you want to create a single or multi-stage environment. In a single-stage environment a cluster has only one OpenFaaS installation and is either staging, production or something else. In a multi-stage environment a cluster may have several installations of OpenFaaS where each represents a different stage. 

##### Single-stage

Since you have a single stage or environment you can apply the namespaces YAML file from the [faas-netes repo](https://github.com/openfaas/faas-netes). This will create two namespaces:

* openfaas
* openfaas-fn

##### Multiple-stages

In a multiple stage environment you may have OpenFaaS installed several times - one for each stage such as "staging" and "production". 

In this case take a copy of the namespaces YAML file and edit it once per each environment such as:

Staging stage:

```
openfaas-staging
openfaas-staging-fn
```

Production stage:

```
openfaas
openfaas-fn
```

Assume that no suffix means that the environment or stage is for production deployments.

## Configure networking

Whether you need to configure new networking for your OpenFaaS deployments, or integrate into existing systems the following guidelines should be followed.

### Configure Ingress

> See also: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

It is recommended that you use an IngressController and TLS so that traffic between your clients and your OpenFaaS Gateway is encrypted.

You may already have opinions about what IngressController you want to use, the maintainers like to use Nginx given its broad adoption and relative ubiquity.

> See also: [Nginx IngressController](https://github.com/kubernetes/ingress-nginx) 

Heptio Contour also includes automatic retries and additional Ingress extensions which you may find useful:

> See also: [Heptio Contour](https://github.com/heptio/contour)

Notes:

* Check any default timeouts set for your IngressController
* Check any default timeout values set for your cloud LoadBalancer

### TLS

Depending on your infrastructure or cloud provider you may choose to add TLS termination in your LoadBalancer, with some other external software or hardware or in the IngressController's configuration. For public-facing OpenFaaS installations the maintainers recommend the use of [cert-manager from JetStack](https://github.com/jetstack/cert-manager).

### Configure NetworkPolicy

You may want to configure NetworkPolicy to restrict communication between the openfaas Functions namespace and the core components of OpenFaaS. This is dependent on using a network driver which supports NetworkPolicy such as Weavenet.

## NATS Streaming (asynchronous invocations)

If you are using the asynchronous invocations available in OpenFaaS then you may want to ensure high-availability or persistence for NATS Streaming. The default configuration uses in-memory storage. Options include clustering NATS and MySQL.

> See also: [NATS documentation](https://nats.io/documentation/)

## Workload or function guidelines

OpenFaaS supports two types of workloads - a microservice or a function. If you are implementing your own microservice or template then ensure that you are complying with the workload contract as per the docs.

> See also: [workload contract](https://docs.openfaas.com/reference/workloads/)

### Pick your watchdog version

For each function it is recommended to evaluate whether the classic or of-watchdog is best suited to your use-case.

For high-throughput, low-latency operations you may prefer to use the of-watchdog.

> See also: [watchdog](https://docs.openfaas.com/architecture/watchdog/)

### Use a non-root user

Each template authored by the OpenFaaS project already uses a non-root user, but if you have your own templates then ensure that they are running as a non-root user to help mitigate against vulnerabilities. 

### Enable a read-only filesystem

It is recommended that you use a read-only filesystem. You will still be able to write to `/tmp/`

### Use secrets and environmental variables appropriately

* Secrets are for confidential data

    Example: API key, IP address, username

* Environmental variables are for non-confidential configuration data

    Example: verbose logging, runtime mode, timeout values

