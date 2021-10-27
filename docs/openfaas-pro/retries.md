# Retries for asynchronous messages

OpenFaaS Pro offers an upgraded queue-worker which allows for messages to be retried a number of times using an exponential back-off algorithm to mitigate the impact associated with retrying messages.

> Note: This functionality is part of [OpenFaaS Pro](https://openfaas.com/support/).

There are two primary use-cases for retrying asynchronous messages:

* When a downstream API is likely to fail some of the time, and cannot be fixed easily, retrying can work around this problem
* When `max_inflight` has been set on a function such as a web scraper, or memory-intensive function and you do not want to overload the function. When a function cannot accept any more connections due to the inflight setting, it will return a 429 error, which indicates the message can be retried at a later time.

## Installation

You can install the Pro version of the queue worker by editing the values.yaml file of the [OpenFaaS chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

```yaml
# Requires OpenFaaS Pro subscription
queueWorkerPro:
  image: ghcr.io/openfaas/queue-worker-pro:0.1.0-rc4
  enabled: false
  maxRetryAttempts: "10"
  maxRetryWait: "120s"
  initialRetryWait: "10s"
  httpRetryCodes: "429,502,500,504,408"
```

The `initialRetryWait` and `maxRetryWait` values are specified in Golang durations, such as `1s`, `1m`, `1m10s` and so forth.

The `maxRetryAttempts` is the amount of times to try a message before giving up and discarding it.

The `httpRetryCodes` is a comma-separated list of HTTP status codes which the queue worker will retry if observed from a function.

## Usage

To test the retry functionality, you can use our chaos function, which allows a function to be configured to return a canned response, or to timeout with a given duration.


```bash
git clone https://github.com/alexellis/chaos-fn

faas-cli deploy -f chaos-fn.yml
```

Cause the API to start failing with a `500` error code:

```bash

curl -i localhost:8080/function/chaos-fn/set --data-binary '
{	"status": 500,
	"delay": "1s",
    "body": "1 second delay, then 500"}
' --header "Content-type: application/json"

```

Now invoke it, and the function will fail with a 500 error code each time it's invoked by the retry logic.

```bash
curl -i localhost:8080/async-function/chaos-fn -d ""
```

Observe the retrying mechanism with the logs of the queue-worker.

```bash
kubectl logs deploy/queue-worker -n openfaas -f
```

Whenever you like, fix the error by changing the canned HTTP response to "200 OK". This will allow the next retry to complete:

```bash
curl -i localhost:8080/function/chaos-fn/set --data-binary '
{"status": 200,
	"delay": "1ms",
    "body": "1ms second delay, then 200"}
' --header "Content-type: application/json"
```

