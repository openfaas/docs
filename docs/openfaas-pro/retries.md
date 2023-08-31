# Retries for functions

OpenFaaS Pro offers an upgraded queue-worker which allows for messages to be retried a number of times using an exponential back-off algorithm to mitigate the impact associated with retrying messages.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

There are two primary use-cases for retrying asynchronous messages:

* When a downstream API is likely to fail some of the time, and cannot be fixed easily, retrying can work around this problem
* When `max_inflight` has been set on a function such as a web scraper, or memory-intensive function and you do not want to overload the function. When a function cannot accept any more connections due to the inflight setting, it will return a 429 error, which indicates the message can be retried at a later time.

## Installation

You can install the Pro version of the queue worker by editing the values.yaml file of the [OpenFaaS chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

You must also set:

```yaml
openfaasPro: true

queueWorkerPro:
  maxRetryAttempts: "10"
  maxRetryWait: "120s"
  initialRetryWait: "10s"
  httpRetryCodes: "429,502,500,504,408"
```

Timings are specified in Golang durations such as `10s` or `1m`.

* `initialRetryWait` is the amount of time to wait for the first retry, this can be set 0s if desired, or a higher value.
* `maxRetryWait` is the maximum amount of time to wait between retries.
* `maxRetryAttempts` is the amount of times to try sending a message to a function before discarding it.
* `httpRetryCodes` is a comma-separated list of HTTP status codes which the queue worker will retry when received from a function.

## Configure retries per function

The retry configuration can be overridden on a per function basis using annotations on the function:

=== "stack.yml"

    ```yaml
    functions:
      chaos:
        image: alexellis2/chaos-fn:0.1.1
        skip_build: true
        annotations:
          com.openfaas.retry.attempts: "30"
          com.openfaas.retry.codes: "429"
          com.openfaas.retry.min_wait: "5s"
          com.openfaas.retry.max_wait: "1m"
    ```

=== "CLI"

    ```bash
    faas-cli store deploy chaos \
      --annotation com.openfaas.retry.attempts=30 \
      --annotation com.openfaas.retry.codes=429 \
      --annotation com.openfaas.retry.min_wait=5s \
      --annotation com.openfaas.retry.max_wait=1m
    ```

The default value configured on the queue worker is used if an annotation is not specified or if it's value is invalid.

## Usage

To test the retry functionality, you can use our chaos function, which allows a function to be configured to return a canned response, or to timeout with a given duration.

We will configure a testing function to output a 429 HTTP response, which indicates that the function is too busy to process the request. This is one of the core use-cases for retrying an invocation. Once we've observed the retrying functionality, we will update the function to return a 200 response, then we'll see the request complete.

You can also add a long timeout or a different HTTP code, see the testing function's notes on GitHub: [alexellis/chaos-fn](https://github.com/alexellis/chaos-fn)

```bash
faas-cli store deploy chaos
```

Open a separate terminal to observe the retrying mechanism with the logs of the queue-worker.

```bash
kubectl logs deploy/queue-worker -n openfaas -f
```

Configure the chaos-fn to start failing with a `429` error code:

```bash
export CODE=429
curl -i http://127.0.0.1:8080/function/chaos/set --data-binary '
{	"status": '$CODE',
	"delay": "1s",
    "body": "1 second delay, then '$CODE'"}
' --header "Content-type: application/json"
```

You will always see a `202 Accepted` from the `/set` path, if you see an error then check the format of your request. The status code must be an integer.

Invoke the function synchronously, to see that it's returning the expected error code:

```bash
curl -i http://127.0.0.1:8080/function/chaos

HTTP/1.1 429 Conflict
Content-Length: 24
Content-Type: text/plain; charset=utf-8
Date: Tue, 08 Mar 2022 11:37:01 GMT
X-Call-Id: 24ae517c-ad0d-4654-8160-d2a7f7d68fab
X-Duration-Seconds: 1.001162
X-Start-Time: 1646739420463153513

1 second delay, then 429
```

Now invoke the function asynchronously.

```bash
curl -i http://127.0.0.1:8080/async-function/chaos -d "input"
```

The invocation will fail, returning a 429 message. This will be received by the queue-worker which will then retry the request. Check the logs of the queue-worker in the other terminal to see this happening.

Allow it to fail several times, seeing the retry time back off exponentially.

> For undelivered messages that reach maximum retries the response is posted to the [callback url](https://docs.openfaas.com/reference/async/#how-it-works). 

Then, whenever you like, fix the error by changing the canned HTTP response to "200 OK". This will allow the next retry to complete:

```bash
curl -i http://127.0.0.1:8080/function/chaos/set --data-binary '
{"status": 200,
	"delay": "1ms",
    "body": "1ms second delay, then 200"}
' --header "Content-type: application/json"
```

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
