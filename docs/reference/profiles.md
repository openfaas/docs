# Profiles: Advanced Provider Configuration

!!! info "OpenFaaS Standard/for Enterprises"
    Profiles are part of [OpenFaaS Standard/for Enterprises](/openfaas-pro/introduction).

The OpenFaaS design allows it to provide a standard API across several different container orchestration tools: Kubernetes, containerd, and others. These [faas-providers](/architecture/faas-provider) generally implement the same core features and allow your to functions to remain portable and be deployed on _any_ certified OpenFaaS installation regardless of the orchestration layer. However, there are certain workloads or deployments that require more advanced features or fine tuning of configuration. To allow maximum flexibility without overloading the OpenFaaS function configuration, we have introduced the concept of Profiles. This is simply a reserved function annotation that the `faas-provider` can detect and use to apply the advanced configuration.

In some cases, there may be a 1:1 mapping between Profiles and Functions, this is to be expected for TopologySpreadConstraints, Affinity rules. We see no issue with performance or scalability.

In other cases, one Profile may serve more than one function, such as when using a toleration or a runtime class.

Multiple Profiles can be composed together for functions, if required.

> Note: The general design is inspired by [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/) and [IngressClasses](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) in Kubernetes. If you are familiar with Kubernetes, these comparisons may be helpful, but they are not required to understand Profiles in OpenFaaS.

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

Profiles are created in the `openfaas` namespace, so typically will be created and maintained by Cluster Administrators.

## Creating Profiles

Profiles must be pre-created, similar to Secrets, usually by the cluster admin. The OpenFaaS API does not provide a way to create Profiles because they are hyper specific to the orchestration tool.

### Enable Profiles

When installing OpenFaaS on Kubernetes, Profiles use a CRD. This must be installed during or prior to start the OpenFaaS controller. When using the [official Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas) this will happen automatically. Alternatively, you can apply this [YAML](https://raw.githubusercontent.com/openfaas/faas-netes/master/yaml/profile-crd.yml) to install the CRD.

### Available Options

Profiles in Kubernetes work by injecting the supplied configuration directly into the correct locations of the Function's Deployment. This allows us to directly expose the underlying API without any additional modifications. Currently, it exposes the following Pod and Container options from the Kubernetes API.

| Field | Description | Availability |
| ------|-------------|--------------|
| `podSecurityContext` | https://kubernetes.io/docs/tasks/configure-pod-container/security-context/ | Standard/for Enterprises |
| `tolerations` | https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration | Standard/for Enterprises |
| `runtimeClassName` |  https://kubernetes.io/docs/concepts/containers/runtime-class/ | OpenFaaS for Enterprises |
| `topologySpreadConstraints`| https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/ | Standard/for Enterprises |
| `affinity` | https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity | Standard/for Enterprises |
| `dnsPolicy` | https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy | Standard/for Enterprises |
| `dnsConfig` | https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config | Standard/for Enterprises |
| `resources` | https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ | Standard/for Enterprises |

The configuration use the exact options that you find in the Kubernetes documentation.

### Examples

#### Use an Alternative RuntimeClass

!!! info "OpenFaaS for Enterprises"
    This feature is part of the [OpenFaaS for Enterprises](/openfaas-pro/introduction) distribution.

A popular alternative container runtime class is [gVisor](https://gvisor.dev/) that provides additional sandboxing between containers. If you have created a cluster that is using gVisor, you will need to set the `runTimeClass` on the Pods that are created. This is not exposed in the OpenFaaS API, but it can be set via a Profile.

1. Install the latest `faas-netes` release and the CRD. The is most easily done with [`arkade`](https://github.com/alexellis/arkade)
    ```sh
    arkade install openfaas \
      --set openfaasPro=true
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


#### Specify a nodeSelector to schedule functions to specific nodes

This example works for OpenFaaS Standard and OpenFaaS for Enterprises only, but you should consider using TopologySpreadConstraints or Affinity rules instead, which are more versatile.

> "nodeSelector is the simplest recommended form of node selection constraint. You can add the nodeSelector field to your Pod specification and specify the node labels you want the target node to have. Kubernetes only schedules the Pod onto nodes that have each of the labels you specify."

Read the Kubernetes docs: [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)

Label a node as follows:

```yaml
cat <<EOF > kind-nodeselectors.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31112
    hostPort: 31112
- role: worker
- role: worker
EOF

kind create cluster --config kind-nodeselectors.yaml

kubectl label node/kind-worker customer=1
kubectl label node/kind-worker2 customer=2
```

kind-worker will take on functions with a constraint of `customer=1` and kind-worker2 will take on the workloads for customer 2.

Now deploy a function with a nodeselector:

```yaml
cat <<EOF > stack.yml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  customer1-env:
    skip_build: true
    image: ghcr.io/openfaas/alpine:latest
    fprocess: env
    constraints:
    - "customer=1"
  customer2-env:
    skip_build: true
    image: ghcr.io/openfaas/alpine:latest
    fprocess: env
    constraints:
    - "customer=1"
EOF

faas-cli deploy stack.yml
```

Confirm the scheduling:

```
$ kubectl get deploy -o wide -n openfaas-fn

NAME            READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS      IMAGES                           SELECTOR
customer1-env   1/1     1            1           18s    customer1-env   ghcr.io/openfaas/alpine:latest   faas_function=customer1-env
customer2-env   1/1     1            1           18s    customer2-env   ghcr.io/openfaas/alpine:latest   faas_function=customer2-env
```

This will also work if you have several nodes dedicated to a particular customer, just apply the label to each node and add the constraint at deployment time.

You may also want to consider using a taint and toleration to ensure OpenFaaS workload components do not get scheduled to these nodes.


#### Spreading your functions out across different zones for High Availability

The [topologySpreadConstraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) feature of Kubernetes provides a more flexible alternative to Pod Affinity / Anti-Affinity rules for scheduling functions.

> "You can use topology spread constraints to control how Pods are spread across your cluster among failure-domains such as regions, zones, nodes, and other user-defined topology domains. This can help to achieve high availability as well as efficient resource utilization."

Imagine a cluster with two nodes, each in a different availability zone.

Let's simulate that with KinD:

```yaml
cat <<EOF > kind-zones.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31112
    hostPort: 31112
- role: worker
- role: worker
EOF

kind create cluster --config kind-zones.yaml

kubectl label node/kind-worker topology.kubernetes.io/zone=a
kubectl label node/kind-worker2 topology.kubernetes.io/zone=b
```

Deploy a profile called `env-tsc`

```yaml
kind: Profile
apiVersion: openfaas.com/v1
metadata:
    name: env-tsc
    namespace: openfaas
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        faas_function: env
```

Deploy a function with this profile:

```bash
faas-cli store deploy env \
  --annotation com.openfaas.profile=tsc
```

Scale the function to 6 replicas:

```bash
kubectl scale -n openfaas-fn deploy/env --replicas=6
```

Notice how the pods are spread evenly between the nodes in the two zones:

```bash
kubectl get pod -n openfaas-fn -o wide -w
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
env-5f5f697594-5clh6   1/1     Running   0          11s   10.244.2.13   kind-worker    <none>           <none>
env-5f5f697594-6fxhq   1/1     Running   0          21s   10.244.2.12   kind-worker    <none>           <none>
env-5f5f697594-92ntn   1/1     Running   0          11s   10.244.1.7    kind-worker2   <none>           <none>
env-5f5f697594-bz6sb   1/1     Running   0          11s   10.244.1.8    kind-worker2   <none>           <none>
env-5f5f697594-hsx98   1/1     Running   0          21s   10.244.1.6    kind-worker2   <none>           <none>
env-5f5f697594-nqxbh   1/1     Running   0          24s   10.244.2.11   kind-worker    <none>           <none>
```

A note on whenUnsatisfiable:

The constraint of `whenUnsatisfiable: DoNotSchedule` will mean pods are not scheduled if they cannot be balanced evenly. This may become an issue for you if your nodes are of difference sizes, therefore you may also want to consider changing this value to `ScheduleAnyway`

#### Use Tolerations and Affinity to Separate Workloads

This example is for OpenFaaS Pro because it uses Affinity.

While the OpenFaaS API exposes the Kubernetes `NodeSelector` via [`constraints`](/reference/yaml#function-constraints), affinity/anti-affinity and taint/tolerations can be used to further expand the types of constraints you can express. OpenFaaS Profiles allow you to set these options. They allow you to more accurately isolate workloads, keep certain workloads together on the same nodes, or to keep certain workloads separate.

For example, a mixture of taints and affinity can put less critical functions on [preemptable vms](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms) that are cheaper while keeping critical functions on standard nodes with higher availability guarantees.

In this example, we create a Profile using taints and affinity to place functions on the node with NVME storage. We will also ensure that _only_ functions that require NVME are scheduled on these nodes. This ensures that the functions that need to faster storage are not blocked by other standard functions taking resources on these special nodes.

1. Install the latest `faas-netes` release and the CRD. The is most easily done with [`arkade`](https://github.com/alexellis/arkade)

    ```sh
    arkade install openfaas \
      --set openfaasPro=true
    ```

    This default installation will enable Profiles.

2. Label _and_ Taint the node with the NVME

    ```sh
    kubectl label nodes <NODE_NAME> nvme=installed
    kubectl taint nodes <NODE_NAME> nvme:NoSchedule
    ```

3. Create a Profile to that allows functions to run on this node
    ```yaml
    kubectl apply -f- << EOF
    kind: Profile
    apiVersion: openfaas.com/v1
    metadata:
      name: withnvme
      namespace: openfaas
    spec:
        tolerations:
        - key: "nvme"
          operator: "Exists"
          effect: "NoSchedule"
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: nvme
                  operator: In
                  values:
                  - installed
    EOF
    ```

3. Then add this annotation to your stack.yml file:

    ```yaml
    com.openfaas.profile: withnvme
    ```

#### Configure DNS for function pods

There are cases when you might want to set a custom DNS configuration per function instead of using the cluster level DNS settings. For example if you are building a multi tenant functions platform and need different DNS configuration for functions from different tenants. Profiles support setting the `dnsPolicy` and `dnsConfig` for a function pod.

1. Create a profile with a custom DNS configuration.
    In this example we configure custom nameservers.


    ```yaml
    kubectl apply -f- << EOF
    kind: Profile
    apiVersion: openfaas.com/v1
    metadata:
        name: function-dns
        namespace: openfaas
    spec:
        dnsPolicy: None
        dnsConfig:
          nameservers:
            - "8.8.8.8"
            - "1.1.1.1"
    EOF
    ```

2. Deploy a function with a profile annotation to apply the dns profile.

    ```bash
    faas-cli store deploy nslookup --annotation "com.openfaas.profile=function-dns"
    ```

Invoke to `nslookup` function to see it uses the custom nameservers.

```bash
echo openfaas.com | faas-cli invoke nslookup

Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   openfaas.com
Address: 185.199.108.153
Name:   openfaas.com
Address: 185.199.109.153
Name:   openfaas.com
Address: 185.199.111.153
Name:   openfaas.com
Address: 185.199.110.153
```

#### Schedule functions with GPU 

You will have to make sure GPU nodes in your cluster are set up with GPU drivers and run the corresponding device plugin from the GPU vendor.
See the kubernetes documentation for detailed information on [scheduling GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

Once you have installed the plugin, your cluster exposes a custom schedulable resource such as `amd.com/gpu` or `nvidia.com/gpu`. These are not exposed through the resources in the OpenFaaS Function spec but can be applied using Profiles.

Here's an example of a Profile that requests one NVIDIA GPU for a function:

```yaml
kind: Profile
apiVersion: openfaas.com/v1
metadata:
    name: nvidia-gpu
    namespace: openfaas
spec:
  runtimeClass: nvidia
  resources:
    limits:
      nvidia.com/gpu: 1 # requesting 1 GPU
```

Note: `runtimeClass` also needs to be set to use the relevant container runtime if your cluster has multiple runtimes.

Add this profile to the cluster and use the `com.openfaas.profile` annotation to apply the profile to functions that need access to a GPU:

```
com.openfaas.profile: nvidia-gpu
```
