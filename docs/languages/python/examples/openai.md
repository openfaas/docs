Send prompts to the [OpenAI Responses API](https://platform.openai.com/docs/guides/text?lang=python) and return the response. Running this inside an OpenFaaS function lets you trigger AI inference on demand via HTTP, or integrate it into event-driven workflows.

Use-cases:

* Chatbots and conversational interfaces
* Content generation and summarisation
* Adding AI features to existing workflows

This example sends a prompt and returns the full completion. The API key is stored as an [OpenFaaS secret](/reference/secrets/).

## Overview

handler.py:

```python
from openai import OpenAI

# Initialise the client once and reuse it across invocations
# to avoid reading the secret and creating a new client on every request.
client = None

def initClient():
    apiKey = read_secret('openai-api-key')
    return OpenAI(api_key=apiKey)

def handle(event, context):
    global client

    if client is None:
        client = initClient()

    # Send the request body as a user message
    response = client.responses.create(
        model="gpt-5.4-nano",
        input=event.body.decode("utf-8")
    )

    return {
        "statusCode": 200,
        "body": response.output_text
    }

def read_secret(name):
    with open("/var/openfaas/secrets/" + name, "r") as f:
        return f.read().strip()
```

requirements.txt:

```
openai
```

stack.yaml:

```yaml
functions:
  openai-chat:
    lang: python3-http
    handler: ./openai-chat
    image: ttl.sh/openfaas-examples/openai-chat:latest
    secrets:
    - openai-api-key
```

The `openai` package is pure Python, so the Alpine-based `python3-http` template works here.

- The OpenAI client is initialised once on first invocation and reused for subsequent requests, avoiding the overhead of re-reading the secret and re-establishing the HTTP connection on every call.
- The `read_secret` helper reads the API key from `/var/openfaas/secrets/`. OpenFaaS mounts secrets as files at that path at runtime — this is preferred over environment variables as the values are not visible in the process environment or container spec.

## Step-by-step walkthrough

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-http
faas-cli new --lang python3-http openai-chat \
  --prefix ttl.sh/openfaas-examples
```

The example uses the public [ttl.sh](https://ttl.sh) registry — replace the prefix with your own registry for production use.

Update `openai-chat/handler.py` and `openai-chat/requirements.txt` with the code from the overview above.

### Create a secret for the API key

Store your [OpenAI API key](https://platform.openai.com/api-keys) as an OpenFaaS secret. This keeps the key out of environment variables and the function's container image.

Save your API key to `openai-api-key.txt`, then run:

```bash
faas-cli secret create openai-api-key --from-file openai-api-key.txt
```

At runtime, the secret is mounted as a file under `/var/openfaas/secrets/` inside the function container.

### Deploy and invoke

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter openai-chat \
 --tag digest
```

Send a prompt to the function:

```bash
curl http://127.0.0.1:8080/function/openai-chat \
  --data "What is the capital of France?"
```

## Streaming responses

To stream tokens back to the client as they are generated, use the `python3-flask` template instead. Flask lets the handler return a `Response` object backed by a generator, which yields each token as a [Server-Sent Event (SSE)](sse.md).

### Overview

handler.py:

```python
from flask import Response
from openai import OpenAI

# Initialise the client once and reuse it across invocations
# to avoid reading the secret and creating a new client on every request.
client = None

def initClient():
    apiKey = read_secret('openai-api-key')
    return OpenAI(api_key=apiKey)

def handle(req):
    global client

    if client is None:
        client = initClient()

    def generate():
        # Request a streaming response from OpenAI
        with client.responses.stream(
            model="gpt-5.4-nano",
            input=req,
        ) as stream:
            # Yield each text delta as an SSE event
            for event in stream:
                if event.type == "response.output_text.delta":
                    yield f"data: {event.delta}\n\n"

        yield "data: [DONE]\n\n"

    # Return a streaming Flask response
    return Response(generate(), mimetype='text/event-stream')

def read_secret(name):
    with open("/var/openfaas/secrets/" + name, "r") as f:
        return f.read().strip()
```

requirements.txt:

```
openai
```

stack.yaml:

```yaml
functions:
  openai-stream:
    lang: python3-flask
    handler: ./openai-stream
    image: ttl.sh/openfaas-examples/openai-stream:latest
    secrets:
    - openai-api-key
```

The `generate()` inner function yields each text delta as an SSE `data:` event, with a final `[DONE]` event to signal the end of the stream. The same API key secret is reused from the non-streaming example.

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-flask
faas-cli new --lang python3-flask openai-stream \
  --prefix ttl.sh/openfaas-examples
```

Update `openai-stream/handler.py` and `openai-stream/requirements.txt` with the code from the overview above.

### Deploy and invoke

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter openai-stream \
 --tag digest
```

Send a prompt and stream the response. The `-N` flag disables curl's output buffering so tokens appear as they arrive. The `Accept: text/event-stream` header tells the OpenFaaS gateway to stream the response instead of buffering it:

```bash
curl -N http://127.0.0.1:8080/function/openai-stream \
  -H "Accept: text/event-stream" \
  -H "Content-Type: text/plain" \
  -d "Explain what SSE is in two sentences."
```

You should see tokens appear incrementally as OpenAI generates them:

```
data: Server
data: -Sent
data:  Events
data:  (
data: SSE
data: )
...
data: [DONE]
```

See also: [Stream Server-Sent Events (SSE)](sse.md) for the general SSE pattern, or [Stream OpenAI responses from functions using Server Sent Events](https://www.openfaas.com/blog/openai-streaming-responses/) on the OpenFaaS blog.
