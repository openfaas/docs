# Scheduling function runs

If you are deploying OpenFaaS to [Kubernetes][k8s], then we can easily run functions as cron jobs using the aptley named [Cron Job resource][k8scron].

We assume that you have used the [recommended install of `faas-netes`][faasdeploy] which menas that you have OpenFaaS deployed into two namespaces:

1.  `openfaas` for the core componentes (ui, gateway, etc)
2.  `openfaas-fn` for the function deployments

## Simple Cron Job

For this example, we use the [sample `nodeinfo` function][nodeinfo], which can be deployed using this stack file

```yaml
# stack.yaml
provider:
  name: faas
  gateway: http://gateway.openfaas.local

functions:
  nodeinfo:
    lang: dockerfile
    handler: node main.js
    image: functions/nodeinfo:latest
    skip_build: true
```

and the cli

```yaml
$ faas deploy
```

We can then define a Kubernetes cron job to call this function every minute using this manifest file:

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
          - name: faas-cli
            image: openfaas/faas-cli:0.6.9
            args:
            - /bin/sh
            - -c
            - echo "verbose" | ./faas-cli invoke nodeinfo -g http://gateway:8080
          restartPolicy: OnFailure
```

The important thing to notice is that we are using a Docker container with the `faas-cli` to invoke the function. This keeps the job very generic and easy to generize to other functions.

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

Unfortunately, there is no one-line command in `kubectl` for getting the logs from a cron job. Kuberenets creates new Job objects for each run of the CronJob, so we can look up that last run of our CronJob using

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

## Other considerations

This simple example uses the default deployment of OpenFaaS without any authentication.

### Multiple Namespaces

In this example, I created the CronJob in the same namespace as the `gateway`. If we deploy the CronJob in a different namespace, then we need to update the job arguments to accomidate. Fortunately, with Kubernetes DNS, this is simply changing the gateway parameter like this `./faas-cli invoke nodeinfo -g http://gateway.othernamespace:8080`

### Authentication

If you have enabled basic auth on the gateway, then the invoke command will also need to be updated to first login the cli client. Assuming that you have created a secret with a file for the username and a separate file for the password, like this

```sh
$ kubectl -n openfaas create secret generic faas-basic-auth --from-file=./username --from-file=./password
```

You could then update the CronJob to login, like this

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
          - name: faas-cli
            image: openfaas/faas-cli:0.6.9
            env:
              - name: FAAS_USERNAME
                valueFrom:
                  secretKeyRef:
                    name: faas-basic-auth
                    key: username
              - name: FAAS_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: faas-basic-auth
                    key: password
            args:
            - /bin/sh
            - -c
            - ./faas-cli login -g http://gateway:8080 -u $FAAS_USERNAME -p $FAAS_PASSWORD
            - echo "verbose" | ./faas-cli invoke nodeinfo -g http://gateway:8080
          restartPolicy: OnFailure
```

[k8s]: https://kubernetes.io/ "Kubernetes"
[k8scron]: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/ "Kuberenetes CRON Jobs"
[k8sjob]: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/ "Kuberenetes Jobs"
[faasdeploy]: https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#deploy-openfaas
[nodeinfo]: https://github.com/openfaas/faas/tree/master/sample-functions/NodeInfo
