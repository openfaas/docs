# Scheduling function runs

## Kubernetes

If you are deploying OpenFaaS to [Kubernetes][k8s], then we can easily run functions as cron jobs using the aptley named [Cron Job resource][k8scron].

We assume that you have used the [recommended install of `faas-netes`][faasdeploy] which means that you have OpenFaaS deployed into two namespaces:

1.  `openfaas` for the core components (ui, gateway, etc)
2.  `openfaas-fn` for the function deployments

### Simple Cron Job

For this example we'll deploy a function which can print system info about the container it's running in:

```bash
faas-cli store deploy nodeinfo
```

We can then define a [Kubernetes cron job](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) to call this function every minute using this manifest file:

```yaml
# node-cron.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: nodeinfo
  namespace: openfaas
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: openfaas-cli
            image: openfaas/faas-cli:latest
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            args:
            - -c
            - echo "verbose" | faas-cli invoke nodeinfo -g http://gateway.openfaas:8080
          restartPolicy: OnFailure
```

You should also update the `image` to the latest version of the `faas-cli` available found via the [Docker Hub](https://hub.docker.com/r/openfaas/faas-cli/tags/) or [faas-cli releases](https://github.com/openfaas/faas-cli/releases) page.

The important thing to notice is that we are using a Docker container with the `faas-cli` to invoke the function. This keeps the job very generic.

We schedule the job by applying our manifest

```sh
$ kubectl apply -f node-cron.yaml

$ kubectl -n=openfaas get cronjob nodeinfo --watch
NAME       SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
nodeinfo   */1 * * * *   False     0         <none>          42s
nodeinfo   */1 * * * *   False     1         2s        44s
nodeinfo   */1 * * * *   False     0         12s       54s
nodeinfo   */1 * * * *   False     1         2s        1m
nodeinfo   */1 * * * *   False     0         12s       1m
```

Unfortunately, there is no one-line command in `kubectl` for getting the logs from a cron job. Kubernetes creates new Job objects for each run of the CronJob, so we can look up that last run of our CronJob using

```sh
$ kubectl -n openfaas get job
NAME                  DESIRED   SUCCESSFUL   AGE
nodeinfo-1529226900   1         1            6s
```

We can use this to then get the output logs

```sh
$ kubectl -n openfaas logs -l "job-name=nodeinfo-1529226900"
Hostname: nodeinfo-6fffdb4446-57mzn

Platform: linux
Arch: x64
CPU count: 1
Uptime: 997420
[ { model: 'Intel(R) Xeon(R) CPU @ 2.20GHz',
    speed: 2199,
    times:
     { user: 360061300,
       nice: 2053900,
       sys: 142472900,
       idle: 9425509300,
       irq: 0 } } ]
{ lo:
   [ { address: '127.0.0.1',
       netmask: '255.0.0.0',
       family: 'IPv4',
       mac: '00:00:00:00:00:00',
       internal: true },
     { address: '::1',
       netmask: 'ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff',
       family: 'IPv6',
       mac: '00:00:00:00:00:00',
       scopeid: 0,
       internal: true } ],
  eth0:
   [ { address: '10.4.2.40',
       netmask: '255.255.255.0',
       family: 'IPv4',
       mac: '0a:58:0a:04:02:28',
       internal: false },
     { address: 'fe80::f08e:d8ff:fecc:9635',
       netmask: 'ffff:ffff:ffff:ffff::',
       family: 'IPv6',
       mac: '0a:58:0a:04:02:28',
       scopeid: 3,
       internal: false } ] }
```

This example assumes no authentication is enabled on the gateway.

### Multiple Namespaces

In this example, I created the CronJob in the same namespace as the `gateway`. If we deploy the CronJob in a different namespace, then we need to update the job arguments to accommodate. Fortunately, with Kubernetes DNS, this is simply changing the gateway parameter like this `./faas-cli invoke nodeinfo -g http://gateway.othernamespace:8080`

### Authentication

If you have enabled basic auth on the gateway, then the invoke command will also need to be updated to first login the cli client. Assuming that you have created the basic auth secret as in the [Helm install guide](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#secure-the-gateway-administrative-api-and-ui-with-basic-auth)

You could then update the CronJob to login, like this:

```yaml
# nodeauth-cron.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: nodeinfo-auth
  namespace: openfaas
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: openfaas-cli
            image: openfaas/faas-cli:latest
            env:
              - name: USERNAME
                valueFrom:
                  secretKeyRef:
                    name: basic-auth
                    key: basic-auth-user
              - name: PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: basic-auth
                    key: basic-auth-password
            command:
            - /bin/sh
            args:
            - -c
            - echo -n $PASSWORD | faas-cli login -g http://gateway.openfaas:8080 -u $USERNAME --password-stdin
            - echo "verbose" | faas-cli invoke nodeinfo -g http://gateway.openfaas:8080
          restartPolicy: OnFailure
```

[k8s]: https://kubernetes.io/ "Kubernetes"
[k8scron]: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/ "Kuberenetes CRON Jobs"
[k8sjob]: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/ "Kuberenetes Jobs"
[faasdeploy]: https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#deploy-openfaas
[nodeinfo]: https://github.com/openfaas/faas/tree/master/sample-functions/NodeInfo

## Cron Connector

The [cron-connector](https://github.com/openfaas-incubator/cron-connector) is an OpenFaaS event-connector which can be used to trigger functions on a timed-basis. It makes use of the OpenFaaS REST API, so it is capable of working with all OpenFaaS providers.

### Kubernetes

* Deploy the connector

```sh
arkade install cron-connector
```

* Now annotate a function with a `topic` of `cron-function` and a `schedule` using a valid CRON expression:

```yaml
# (Abridged YAML)

functions:
  nodeinfo:
    image: functions/nodeinfo
    skip_build: true
    annotations:
      topic: cron-function
      schedule: "*/5 * * * *"
```
*nodeinfo.yaml*

```sh
faas-cli deploy -f nodeinfo.yaml
```

* Or deploy directly from the store

```sh
faas-cli store deploy nodeinfo \
  --annotation topic="cron-function" \
  --annotation schedule="*/5 * * * *"
```

* Now check the logs

```sh
kubectl logs -n openfaas-fn deploy/nodeinfo -f
```

You'll see the function invoked every 5 minutes as per the schedule.

To stop the invocations, remove the two annotations or remove the cron-connector deployment.

If you would like to explore how to write CRON expressions, then see [https://crontab.guru/](https://crontab.guru/)

## Docker Swarm

Docker Swarm has no concepts of scheduled tasks or cron, but we have a suitable recommendation which you can use with your OpenFaaS cluster. If you deploy a Jenkins master service, then you can use that to manage your scheduled tasks. It will handle distributed locking, concurrency and queueing.

Example usage:

* Deploy Swarm service for Jenkins using [Official Docker Hub image](https://hub.docker.com/r/jenkins/jenkins/)
* Define a Freestyle job for each scheduled task
* Add a CRON entry for the schedule
* Install the OpenFaaS CLI
* Run `faas-cli login --gateway`
* Invoke the function

Here is an example of how to do this with a [Pipeline job](https://gist.github.com/alexellis/dfa1b8790ac3d26614e342746c64cbc8).

Alternatively see the above cron-connector example.
