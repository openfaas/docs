# Deploy OpenFaaS Pro in Production

OpenFaaS Pro is meant for production.

This page contains recommendations for tuning OpenFaaS Pro for production usage.

!!! info "Did you know?"
    We offer an initial onboarding call for OpenFaaS Pro users, or a more detailed review as part of our Enterprise Support package.
    
    [Talk to us](https://openfaas.com/pricing/)

## Decide on your Kubernetes flavour

OpenFaaS Pro requires [Kubernetes](https://kubernetes.io/).

You can either use a managed Kubernetes service from a public cloud provider such as GKE, AWS EKS or Digital Ocean Kubernetes, or deploy to your own infrastructure in the cloud or privately, on-premises using something like K3s, Rancher, or OpenShift.

These instructions apply for both Kubernetes, however OpenShift 3.x and 4.x are compatible with some additional tuning, which you can request via support.

OpenFaaS Standard and For Enterprises can also be installed into an airgapped Kubernetes cluster using tooling of your choice, or our own internal tooling [airfaas]()

### Advanced Kubernetes configuration

* [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) (optional)

    The default configuration for Kubernetes is to store secrets in base64 which is equivalent to plaintext and is not a form of encryption. Kubernetes supports encrypting secrets at rest and because OpenFaaS uses Kubernetes secrets, its secrets will also be encrypted at rest.

### Helm Charts

The `helm` chart is the most flexible way to deploy OpenFaaS and allows you to customize many different options such as whether scale-to-zero is enabled, how many replicas to have of each component and what kind of healthchecks or probes to use.

You will find each helm chart option documented in the [OpenFaaS Chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#configuration).

* [Setup instructions are available here for OpenFaaS Pro - Standard and For Enterprises](https://docs.openfaas.com/deployment/pro/), including suggested values for the Helm chart.

The chart is compatible with [GitOps-style](https://www.weave.works/technologies/gitops/) tooling such as [Flux](https://fluxcd.io/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/). We recommend using one of these tools to manage your deployment of OpenFaaS declaratively.

Note that some optional components such as the Kafka event-connector and cron-connector have their own separate helm charts.

### Take advantage of the OpenFaaS Pro add-ons

Check out the overview to see what is available to you under your license:

* [OpenFaaS Pro Overview](https://docs.openfaas.com/openfaas-pro/introduction/)

#### High-availability (HA)

It is recommended that you have high-availability for the OpenFaaS components including:

* OpenFaaS gateway

    For a production environment, we recommend a minimum of three replicas of the gateway. 

    By default, the OpenFaaS gateway includes a Topology Spread Constraints to ensure that replicas are spread across different nodes. This means that if a node were to be reclaimed or any reason, the gateway would still be available, since at least one replica would be running on another node

    Optionally, you can set an additional auto-scaling rule with HPAv2 to scale the gateway automatically based on CPU and memory usage. The percentage values are based upon the requests and limits set for the gateway pod, and should be tuned to suit your own workload:

    ```yaml
    # 50% CPU or 50% RAM is only a guide, and you need to 
    # tune this to suit your own needs
    
    # 3 replicas are required for HA

    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: gateway-hpa
      namespace: openfaas
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: gateway
      minReplicas: 3
      maxReplicas: 6
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
      - type: Resource
        resource:
          name: memory
          target:
            type: Utilization
            averageUtilization: 50
    ```

* OpenFaaS queue-worker

    The queue worker is responsible for processing asynchronous invocations. It's recommended to run at least two replicas of this service. Each queue worker can potentially run hundreds of invocations concurrently, and can be configured with different queues per function if required.

* NATS JetStream

    The default configuration for NATS JetStream uses in-memory storage which means if the NATS Pod crashes or is relocated to another node, you will lose messages. It is recommended to turn off NATS in the OpenFaaS chart and to install it separately using the NATS Operator or helm chart using a Persistent Volume, high-availability, or both. Details are available in the [customer community](https://github.com/openfaas/customers)

* Prometheus

    The Prometheus instance is used to scrape metrics from the OpenFaaS components and functions. It is designed to be ephemeral and uses storage within the Pod, which means if the Pod is restarted, you will lose metrics. In practice this is not a problem, since the metrics required to operate only need to be short-lived.

    It is not supported for OpenFaaS itself to use an external Prometheus instance, so if you need long-term retention of metrics, you need to run a second Prometheus instance which scrapes the internal one. This is known as "federation"

* AlertManager

    AlertManager is not used in OpenFaaS Pro

See the [getting started page for OpenFaaS Pro](https://docs.openfaas.com/deployment/pro/) for more options on the above.

#### Endpoint load-balancing

Endpoint load-balancing is where OpenFaaS distributes load between functions using the endpoints of each function replica in the cluster.

You should use this as a default, but if you use a service mesh you will need to turn it off because a service mesh manages this for you using its own proxy.

* [Endpoint load-balancing](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#endpoint-load-balancing)

#### Configure timeouts

Configure each timeout as appropriate for your workloads. If you expect all functions or endpoints to return within a few seconds then set a timeout to accommodate that. If you expect all workloads to run for several minutes then bear that in mind with your values.

> Note: You may have a timeout configured in your IngressController or cloud LoadBalancer. See your documentation or contact your administrator to check the value and make sure they are in-sync.

See also: [Expanded timeouts](/tutorials/expanded-timeouts/)

#### Configure function health-check probes

There are two types of health-check probes: exec and http. The default in the helm chart is exec for backwards-compatibility.

It is recommended to use httpProbes by default which are more CPU efficient than exec probes. If you are using [Istio](https://istio.io) with mutual TLS then you will need to use exec probes.

#### Configure function health-check interval

There are both liveness and readiness checks for functions. If you are using the scale-from-zero feature then you may want to set these values as low as possible, however this may require additional CPU. A sensible default would be something that makes sense for your own use-case. Tune each value as needed.

#### Non-root user for functions

Set the non-root user flag so that every function is forced to run in a restricted security context.

#### Use only the operator and CRD

Previously, a "controller" mode was available for OpenFaaS which had an imperative REST API and no CRD.

This was replaced by the "operator" mode, which should always be used in production. The operator mode uses a Custom Resource Definition (CRD) to manage functions and other resources.

The setting needs to be: `operator.create: true`

See the [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas) for more.

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

Find out more here: [Configure your OpenFaaS functions for staging and production](https://www.openfaas.com/blog/custom-environments/)

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

### Pin IP addresses with floating IPs (optional)

Most cloud providers allow users to allocate a permanent IP address or Virtual/Floating IP to provisioned LoadBalancers. It is recommended that you do this for production usage. For AWS EKS, this doesn't apply due to the use of a CNAME for LoadBalancers.

### TLS

Depending on your infrastructure or cloud provider you may choose to add TLS termination in your LoadBalancer, with some other external software or hardware or in the IngressController's configuration. For public-facing OpenFaaS installations the maintainers recommend the use of [cert-manager from JetStack](https://github.com/jetstack/cert-manager).

### mTLS

There was a conscious decision made not to bake a service mesh into OpenFaaS, however it is very easy to add one to enable mutual TLS between functions and the gateway.

You can enable mutual TLS (mTLS) between OpenFaaS services using Linkerd or Istio, the community prefers Linkerd for its low resource consumption and ease of use.

* [Tutorial/Lab: Linkerd2 & OpenFaaS](https://github.com/openfaas-incubator/openfaas-linkerd2)

### Reduce DNS lookups when calling other functions

By default, Kubernetes will search for a name like `gateway` with every domain and local domain extension within `/etc/hosts`, which could lead to dozens of DNS lookups per outgoing HTTP call from a function.

To make this most efficient, always address the gateway using its complete name:

```
http://gateway.openfaas.svc.cluster.local:8080/function/function-name
```

The suffix is usually `svc.cluster.local`, however, this can vary between clusters.

### Configure NetworkPolicy

You may want to configure NetworkPolicy to restrict communication between the openfaas Functions namespace and the core components of OpenFaaS.

This is to prevent unauthorized or unintended access to the core components of OpenFaaS (the control plane) by functions deployed by your users.

This is dependent on using a network driver which supports NetworkPolicy such as [Weave Net](https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/weave-network-policy/) or [Calico](https://projectcalico.docs.tigera.io/security/calico-network-policy).

See also: [an example from OpenFaaS Cloud](https://github.com/openfaas/openfaas-cloud/tree/master/yaml/network-policy)

## NATS JetStream (asynchronous invocations)

If you are using the asynchronous invocations available in OpenFaaS then you may want to ensure high-availability or persistence for NATS Streaming. The default configuration uses in-memory storage which means if the NATS Pod crashes or is relocated to another node, you may lose messages.

The easiest way to make NATS durable for production is to turn it off in the OpenFaaS chart, and to install it separately using the NATS Operator or helm chart using a Persistent Volume or SQL as a backend. 

> See also: [NATS documentation](https://nats-io.github.io/docs/)

## Workload or function guidelines

OpenFaaS supports two types of workloads - a microservice or a function. If you are implementing your own microservice or template then ensure that you are complying with the workload contract as per the docs.

> See also: [workload contract](https://docs.openfaas.com/reference/workloads/)

### Pick your watchdog version

For each function it is recommended to evaluate whether the classic or of-watchdog is best suited to your use-case.

For high-throughput, low-latency operations you may prefer to use the of-watchdog.

> See also: [watchdog](https://docs.openfaas.com/architecture/watchdog/)

### Use a non-root user

Each template authored by the OpenFaaS Project already uses a non-root user, but if you have your own templates then ensure that they are running as a non-root user to help mitigate against vulnerabilities. 

### Enable a read-only filesystem

It is recommended that you use a read-only filesystem. You will still be able to write to `/tmp/`

### Use secrets and environmental variables appropriately

Find out more here: [Configure your OpenFaaS functions for staging and production](https://www.openfaas.com/blog/custom-environments/)

* Secrets are for confidential data

    Example: API key, IP address, username

* Environmental variables are for non-confidential configuration data

    Example: verbose logging, runtime mode, timeout values

## Reach out for a review

!!! info "Did you know?"
    We offer an initial onboarding call for OpenFaaS Pro users, or a more detailed review as part of our Enterprise Support package.
    
    [Talk to us](https://openfaas.com/pricing/)
