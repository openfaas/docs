# Profiles: Advanced Provider Configuration

> Note: Only the Kubernetes provider currently supports Profiles!

The OpenFaaS design allows it to provide a standard API across several different container ochestration tools: Kubernetes, Docker Swarm, ContainerD, etc. These [faas-providers](/docs/architecture/faas-provider.md) generally implement the same core features and allow your to functions to remain portable and be deployed on _any_ certified OpenFaaS installation regardless of the orchestration layer. However, there are certain workloads or deployments that require more advanced features or fine tuning of configuration. To allow maximum flexibility without overloading the OpenFaaS function configuration, we have introduced the concept of Profiles. This is simply a reserved function annotation that the `faas-provider` can detect and use to apply the advanced configuration.

> Note: The general design is inspired by [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)  and [IngressClasses](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) in Kubernetes. If you are familiar with Kubernetes, these comparisons may be helpful, but they are not required to understand Profiles in OpenFaaS.


## Using Profiles When You Deploy a Function

If you are a function author, using a Profile is a simple as adding an annotation to your function:

```
com.openfaas.profile: <profile_name>
```

You can do this with the `faas-cli` flags:

```sh
faas-cli deploy --annotation com.openfaas.profile=<profile_name>
```

Or in the stack YAML:
```yaml
functions:
  foo:
    image: "..."
    fprocess: "..."
    annotations:
      com.openfaas.profile: <profile_name>
```

If you need multiple profiles, you can use a comma separated value:

```
com.openfaas.profile: <profile_name1>,<profile_name2>
```

> Note: You must ask your cluster administrator which, if any profiles, are available.


## Creating Profiles

Profiles must be pre-created, similar to Secrets, by the cluster admin. The OpenFaaS API does not provide a way to create Profiles because they are hyper specific to the orchestration tool.

Because Kubernetes is the only provider currently supporting Profiles, the follwing walk-through documents the process and options for Kubernetes.

### Enable Profiles

When installing OpenFaaS on Kubernetes, Profiles use a CRD. This must be installed during or prior to start the OpenFaaS controller. When using the [official Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas) this will happen automatically. Alternatively, you can apply this [YAML](https://github.com/openfaas/faas-netes/blob/master/yaml/crd.yml) to install the CRD.


> Note: Profiles requires `faas-netes` version `>= 0.12.0` and/or Helm chart version `>= 6.0.0`.


### Available Options

Profiles in Kubernetes work by injecting the supplied configuration directly into the correct locations of the Function's Deployment. This allows us to directly expose the underlying API without any additional modifications. Currently, it exposes the following Pod and Container options from the Kubernetes API

- `runtimeClassName` : See https://kubernetes.io/docs/concepts/containers/runtime-class/ for a description and links to any additional documentation about Pod Runtime Class
- `tolerations` : https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/ for a description and links to any additional documentation about Tolerations.
- `affinity` : https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity for a description and links to any additional documentation about Node Affinity.
- `podSecurityContext` : https://kubernetes.io/docs/tasks/configure-pod-container/security-context/ for a description and links to any additional documentation about the Pod Security Context.

The configuration use the exact options that you find in the Kubernetes documentation.

### Examples

#### Use an Alternative RuntimeClass
A popular alternative container runtime class is [gVisor](https://gvisor.dev/) that provides additional sandboxing between containers. If you have created a cluster that is using gVisor, you will need to set the `runTimeClass` on the Pods that are created. This is not exposed in the OpenFaaS API, but it can be set via a Profile.

1. Install the latest `faas-netes` release and the CRD. The is most easily done with [`arkade`](https://github.com/alexellis/arkade)
    ```sh
    arkade install openfaas
    ```
    This default installation will enable Profiles.
2. Create a Profile to apply the runtime class
    ```yaml
    kubectl apply -f- << EOF
    kind: Profile
    apiVersion: openfaas.com/v1
    metadata:
        name: gvisor
        namespace: openfaas
    spec:
        runtimeClassName: gvisor
    EOF
    ```
3. Let your developers know that they need to use this annotation
    ```
    com.openfaas.profile: gvisor
    ```

The following stack file will deploy a SHA512 generating file in a cluster with gVisor

```yaml
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  stronghash:
    skip_build: true
    image: functions/alpine:latest
    fprocess: "sha512sum"
    annotations:
      com.openfaas.profile: gvisor
```

#### Use Tolerations and Affinity to Separate Workloads
The OpenFaaS API exposes the Kubernetes `NodeSelector` via [`constraints`](/docs/reference/yaml#function-constraints). This provides a very simple selection based on labels on Nodes.

The Kubernetes API also exposes two features affinity/anti-affinity and taint/tolerations that further expand the types of constraints you can express.  OpenFaaS Profiles allow you to set these options, allowing you to more accurately isolate workloads, keep certain workloads together on the same nodes, or to keep certain workloads separate.

For example, a mixture of taints and affinity can put less critical functions on [preemptable vms](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms) that are cheaper while keeping critical functions on standard nodes with higher availability guarantees.

In this example, we create a Profile using taints and affinity to place functions on the node with a GPU.  We will also ensure that _only_ functions that require the GPU are scheduled on these nodes. This ensures that the functions that need to use the GPU are not blocked by other standard functions taking resources on these special nodes.

1. Install the latest `faas-netes` release and the CRD. The is most easily done with [`arkade`](https://github.com/alexellis/arkade)
    ```sh
    arkade install openfaas
    ```
    This default installation will enable Profiles.

2. Label _and_ Taint the node with the GPU
    ```sh
    kubectl labels nodes node1 gpu=installed
    kubectl taint nodes node1 gpu:NoSchedule
    ```

3. Create a Profile to that allows functions to run on this node
    ```yaml
    kubectl apply -f- << EOF
    kind: Profile
    apiVersion: openfaas.com/v1
    metadata:
      name: withgpu
      namespace: openfaas
    spec:
        tolerations:
        - key: "gpu"
          operator: "Exists"
          effect: "NoSchedule"
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                - key: gpu
                  operator: In
                  values:
                  - installed
    EOF
    ```
3. Let your developers creating functions that need GPU support, they must use this annotation
    ```
    com.openfaas.profile: withgpu
    ```
