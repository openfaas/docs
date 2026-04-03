Stream data from a Python function to a client as it becomes available using [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events).

Use-cases:

* Streaming LLM completions token by token
* Progress updates for long-running tasks
* Real-time log tailing

This example uses the `python3-flask` template, which lets the handler return a Flask `Response` object with a generator function.

!!! note "Required client header"
    When invoking the function, clients must include `Accept: text/event-stream` so the OpenFaaS gateway streams the response instead of buffering it.

## Overview

handler.py:

```python
import time
from flask import Response

def handle(req):
    def generate():
        for i in range(1, 6):
            time.sleep(1)
            yield f"data: Message {i} of 5\n\n"
        yield "data: [DONE]\n\n"

    return Response(generate(), mimetype='text/event-stream')
```

stack.yaml:

```yaml
functions:
  sse-example:
    lang: python3-flask
    handler: ./sse-example
    image: ttl.sh/openfaas-examples/sse-example:latest
```

No additional pip dependencies are needed — Flask is included in the `python3-flask` template.

!!! info "About the python3-flask template"

    The `python3-flask` template exposes a simpler handler interface than the `python3-http` template. The handler receives the raw request body as a string, and can return a string, a tuple of `(body, status_code)`, a tuple of `(body, status_code, headers)`, or a Flask `Response` object.

## Step-by-step walkthrough

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-flask
faas-cli new --lang python3-flask sse-example \
  --prefix ttl.sh/openfaas-examples
```

The example uses the public [ttl.sh](https://ttl.sh) registry — replace the prefix with your own registry for production use.

Update `sse-example/handler.py` with the code from the overview above.

### Deploy and invoke

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter sse-example \
 --tag digest
```

Stream events from the function:

```bash
curl -N http://127.0.0.1:8080/function/sse-example \
  -H "Accept: text/event-stream"
```

You should see each message appear one second apart:

```
data: Message 1 of 5

data: Message 2 of 5

data: Message 3 of 5

data: Message 4 of 5

data: Message 5 of 5

data: [DONE]
```

!!! note "Timeouts"

    Streaming responses can run for longer than the default function timeout. Make sure your OpenFaaS [timeout values](/tutorials/expanded-timeouts/) are configured appropriately for your streaming workloads.

See also: [OpenAI Chat API](openai.md#streaming-responses) for a practical example that streams LLM completions token by token using this same pattern.
