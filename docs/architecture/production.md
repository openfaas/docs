# Deploy OpenFaaS in Production

This page contains recommendations for deploying OpenFaaS for production usage. It is by no means exhaustive and comes with no warranty or liability.

## Decide on your Kubernetes flavour

When deploying OpenFaaS it is recommended that you use Kubernetes. A managed Kubernetes service such as GKE, EKS is recommended if you are using public cloud. You can also deploy to your own infrastructure.

### Chart options

The `helm` chart is the most flexible way to deploy OpenFaaS and allows you to customize many different options such as whether scale-to-zero is enabled, how many replicas to have of each component and what kind of healthchecks or probes to use.

Depending on your existing usage of `helm` or your preferences you may not want to use the *Tiller* component of helm. In this case you can use `helm template` in the deployment guide to generate YAML files to apply on your cluster.

#### High-availability (HA)

It is recommended that you have high-availability for the OpenFaaS components including:

* OpenFaaS gateway
* OpenFaaS queue-worker

You may also want to set a minimum availability level for each function of > 1.

See the chart options for how to set the number of replicas.

#### Configure timeouts

Configure each timeout as appropriate for your workloads. If you expect all functions or endpoints to return within a few seconds then set a timeout to accommodate that. If you expect all workloads to run for several minutes then bear that in mind with your values. 

#### Configure function health-check probes

Unless you are using Istio, where it is essential to use exec health-checks, it is recommended to use httpProbes by default which are more efficient on CPU.

#### Configure function health-check interval

There are both liveness and readiness checks for functions. If you are using the scale-from-zero feature then you may want to set these values as low as possible, however this may require additional CPU. A sensible default would be something that makes sense for your own use-case. Tune each value as needed.

#### Non-root user for functions

Set the non-root user flag so that every function is forced to run in a restricted security context.

#### Chose the operator or faas-netes

The faas-netes controller for OpenFaaS is the most mature, well tested and supported. You may also use the OpenFaaS Operator if you would like to use a Function CRD. If you are not sure which to use, then use faas-netes.

#### Single or multi-stage?

Now decide if you want to create a single or multi-stage environment. In a single-stage environment a cluster has only one OpenFaaS installation and is either staging, production or something else. In a multi-stage environment a cluster may have several installations of OpenFaaS where each represents a different stage. 

##### Single-stage

Since you have a single stage or environment you can apply the namespaces YAML file from the faas-netes repo. This will create two namespaces:

* openfaas
* openfaas-fn

##### Multiple-stages

In a multiple stage environment you may have OpenFaaS installed several times - one for each stage such as "staging" and "production". 

In this case take a copy of the namespaces YAML file and edit it once per each environment such as:

```
openfaas-staging
openfaas-staging-fn
openfaas
openfaas-fn
```

Assume that no suffix means that the environment or stage is for production deployments.

## Configure Ingress

It is recommended that you use an IngressController and TLS so that traffic between your clients and your OpenFaaS Gateway is encrypted.

You may already have opinions about what IngressController you want to use, the maintainers like to use Nginx given its broad adoption and relative ubiquity.

Notes:

* Check any default timeouts set for your IngressController
* Check any default timeout values set for your cloud LoadBalancer

## TLS

Depending on your infrastructure or cloud provider you may chose to add TLS termination in your LoadBalancer, with some other external software or hardware or in the IngressController's configuration. For public-facing OpenFaaS installations the maintainers recommend the use of [cert-manager from JetStack](https://github.com/jetstack/cert-manager).

## Configure NetworkPolicy

You may want to configure NetworkPolicy to restrict communication between the openfaas Functions namespace and the core components of OpenFaaS. This is dependent on using a network driver which supports NetworkPolicy such as Weavenet.

