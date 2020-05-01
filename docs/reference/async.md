## Asynchronous Functions

OpenFaaS enables long-running tasks or function invocations to run in the background through the use of NATS Streaming. This decouples the HTTP transaction between the caller and the function.

* The HTTP request is  serialized to NATS Streaming through the gateway as a "producer".
* The queue-worker acts as a subscriber and deserializes the HTTP request and uses it to invoke the function directly

The asynchronous workflow can have a longer, separate timeout compared with synchronous timeout on the gateway.

Why might you want to use an asynchronous invocation?

You're working with a partner's webhook. They send you data, but if you don't send a HTTP 200 OK within 1 second, they assume failure and retry the message. This works well if your function can complete in less than a second on every invocation, but if there's any risk that it can't, you need another solution. So you will give your partner a URL to the asynchronous function URL instead and it will reply within several milliseconds whilst still processing the data.

### Synchronous vs asynchronous invocation

* Sync

    A synchronous invocation of a user requesting a PDF would be as follows, where connections are established between each component from the beginning to the end of the invocation.

    ![Synchronous](/images/sync.png)

If the whole process takes less than a few seconds, this may be the ideal approach, given that it's simple to implement.

* Async

    An asynchronous invocation of a user requesting a PDF would be as follows: an initial connection is formed to the gateway, the user's request is serialized to a queue via the queue-worker and NATS. At a later time, the queue-worker then dequeues the request, deserializes it and makes it to the function - either directly or via the gateway using a synchronous call.

    ![Asynchronous](/images/async.png)

    This is beneficial if there are for instance 100 requests that all take 2 minutes to execute. It means the client / caller needs to wait only for a new milliseconds.

### How it works

Any function can be invoked asynchronously by changing the route on the gateway from `/function/<name>` to `/async-function/<name>`. A `202 Accepted` message will be issued in response to asynchronous calls.

> Note: that asynchronous invocations do not make sense with a HTTP `GET` verb since they are queued and deferred, there is nothing to `GET`. For this reason, a HTTP `POST` is required.

If you would like to receive a value from an asynchronous call you should pass a HTTP header with the URL to be used for the call-back.

```
$ faas invoke figlet -H "X-Callback-Url: https://request.bin/mybin"
```

It will pass back the X-Call-Id you had when you sent the initial request.

You can use `netcat` to check the Call Id during invocation:

```
$ curl http://127.0.0.1:8080/async-function/figlet \
  --data "Hi" \
  --header "X-Callback-Url: http://<your-ip>:8888"
```

```
$ nc -l 8888
HTTP/1.1 200 OK
Content-Length: 174
Content-Type: application/x-www-form-urlencoded
Date: Tue, 04 Dec 2018 09:24:55 GMT
X-Call-Id: eb8283f5-1679-48e0-afec-194544b054aa
X-Duration-Seconds: 0.002885
X-Start-Time: 1543915495384346700

 _   _      _ _       
| | | | ___| | | ___  
| |_| |/ _ \ | |/ _ \ 
|  _  |  __/ | | (_) |
|_| |_|\___|_|_|\___/ 

```

Alternatively you can specify another asynchronous or synchronous function to run instead.

#### Parallelism

By default there is one queue-worker replica deployed which is set up to run a single task of up to 30 seconds in duration.

> You can increase the parallelism by scaling the queue-worker up - i.e. 5 replicas for 5 parallel tasks.
>
> Alternatively you can increase the parallelism by setting the queue worker's "max_inflight" option to a value greater than one. This will cause the queue-worker to concurrently receive up to max_inflight many messages and simultaneously invoke their corresponding functions. Should you wish to restrict concurrenty for certain functions, please make use of  [multiple queues](#Multiple-queues) and separate these functions accordingly. When scaling up a queue worker, please be aware that you will get up to 'n * max_inflight' parallel function invocations.

You can tune the values for the number of tasks each queue worker may run in parallel as well as the maximum duration of any asynchronous task that worker processes. Edit the Kubernetes helm chart, YAML or Swarm docker-compose.yml files.

The [OpenFaaS workshop](https://github.com/openfaas/workshop) has more instructions on running tasks asynchronously.

#### Multiple queues

Asynchronous requests are processed by the queue-worker component using a single topic (`faas-request`), for most use-cases this will be sufficient if most of your functions take a similar amount of time to execute. A problem may arise when you have a mixture of slow and fast running requests within the same single queue. A single slow task can hold up all the other requests and this is because the queue has FIFO semantics - first in, first out.

To use multiple queues you need to do two things:

1) Annotate your functions with a `com.openfaas.queue` queue or (topic) name

Imagine that your new queue is called `slow-queue`, you would run the following:

```bash
faas-cli store deploy figlet --annotation com.openfaas.queue=slow-queue
```

2) Create a queue-worker for the new queue name

You now need to deploy a new queue-worker for the queue name, so that it can subscribe to messages and invoke functions without affecting the default queue.

On Kubernetes:

```bash
export CORE_NS=openfaas
kubectl get -n $CORE_NS deploy/queue-worker -o yaml --export > slow-queue-queue-worker.yaml
```

Now replace "queue-worker" with "slow-queue-queue-worker" in `app: queue-worker` and `name: queue-worker`.

```bash
sed -ie s/app:\ queue-worker/app:\ slow-queue-worker/g slow-queue-queue-worker.yaml
sed -ie s/name:\ queue-worker/name:\ slow-queue-worker/g slow-queue-queue-worker.yaml
```

Edit the `faas_nats_channel` environment variable, place `slow-queue` in the `value` field:

```yaml
        - name: faas_nats_channel
          value: slow-queue
```

Deploy the new queue worker for the `slow-queue` queue:

```bash
export CORE_NS=openfaas
kubectl create -f slow-queue-queue-worker.yaml --namespace $CORE_NS
```

You can now invoke your function as per normal and watch the logs of the new queue worker:

```bash
kubectl logs deploy/slow-queue-worker -n openfaas &

curl http://127.0.0.1:8080/async-function/figlet -d "Ran on the slow-queue"
```

#### Verbose Output

The Queue Worker component enables asynchronous processing of function requests. The default verbosity level hides the message content, but this can be viewed by setting write_debug to true when deploying.

#### Callback request headers

The following additional request headers will be set when invoking the call back URL:

| Header             | Description |
|--------------------|-------------|
| X-Call-Id          | The original function call's [tracing UUID](https://github.com/openfaas/faas/blob/master/gateway/README.md#tracing) |
| X-Duration-Seconds | Time taken in seconds to execute the original function call |
| X-Function-Status  | [HTTP status code](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) returned by the original function call |
| X-Function-Name    | The name of the original function that was executed |
