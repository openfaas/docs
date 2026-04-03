[Playwright](https://playwright.dev/python/) is a Python library for controlling a headless browser programmatically. Running it inside an OpenFaaS function lets you trigger browser automation on demand via HTTP — useful for tasks that require a real rendering engine rather than a plain HTTP client.

Use-cases:

* End-to-end and compliance tests against your application
* Scraping information from a webpage that doesn't provide an API
* Taking screenshots or generating PDFs of rendered pages

## Overview

handler.py:

```python
import json
from playwright.sync_api import sync_playwright

def handle(event, context):
    body = json.loads(event.body)
    uri = body.get("uri", "https://www.openfaas.com")

    with sync_playwright() as p:
        browser = p.chromium.launch(
            args=["--no-sandbox", "--disable-setuid-sandbox"]
        )
        try:
            page = browser.new_page()
            page.goto(uri, timeout=30000)
            title = page.title()
        finally:
            browser.close()

    return {
        "statusCode": 200,
        "body": json.dumps({"title": title})
    }
```

requirements.txt:

```
playwright
```

stack.yaml:

```yaml
functions:
  playwright-scrape:
    lang: dockerfile
    handler: ./playwright-scrape
    image: ttl.sh/openfaas-examples/playwright-scrape:latest
    environment:
      PLAYWRIGHT_BROWSERS_PATH: /home/app/.cache/ms-playwright
```

This example uses the `dockerfile` language with a custom Dockerfile based on the `python3-http-debian` template.

- The custom Dockerfile is needed to install the Chromium browser and its system-level dependencies at build time — something the standard template cannot do.
- `PLAYWRIGHT_BROWSERS_PATH` tells Playwright where the Chromium binary was installed inside the image.

## Step-by-step walkthrough

### Create the function

Scaffold the function using the `dockerfile` language:

```bash
faas-cli new --lang dockerfile playwright-scrape \
  --prefix ttl.sh/openfaas-examples
```

Replace the generated Dockerfile with the one below. It is based on the `python3-http-debian` template with an additional step to install Playwright's Chromium browser and its system dependencies.

`playwright-scrape/Dockerfile`:

```dockerfile
ARG PYTHON_VERSION=3.12
ARG DEBIAN_OS=slim-bookworm

FROM --platform=${TARGETPLATFORM:-linux/amd64} ghcr.io/openfaas/of-watchdog:0.11.5 AS watchdog
FROM --platform=${TARGETPLATFORM:-linux/amd64} python:${PYTHON_VERSION}-${DEBIAN_OS} AS build

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

ARG ADDITIONAL_PACKAGE
ARG UPGRADE_PACKAGES=false

RUN apt-get update -qy \
    && if [ "${UPGRADE_PACKAGES}" = "true" ] || [ "${UPGRADE_PACKAGES}" = "1" ]; then apt-get upgrade -qy; fi \
    && apt-get install -qy --no-install-recommends gcc make ${ADDITIONAL_PACKAGE} \
    && rm -rf /var/lib/apt/lists/*

# Add non root user
RUN addgroup --system app \
    && adduser app --system --ingroup app --home /home/app \
    && chown app:app /home/app

USER app

ENV PATH=$PATH:/home/app/.local/bin

WORKDIR /home/app/

COPY --chown=app:app index.py           .
COPY --chown=app:app requirements.txt   .
USER root
RUN pip install --no-cache-dir -r requirements.txt
USER app

RUN mkdir -p function
RUN touch ./function/__init__.py
WORKDIR /home/app/function/
COPY --chown=app:app function/requirements.txt	.
RUN pip install --no-cache-dir --user -r requirements.txt

# Install Chromium and system dependencies for Playwright.
# The PYTHONPATH is set because pip packages are in the app user's
# local site-packages, but this step runs as root.
USER root
ENV PLAYWRIGHT_BROWSERS_PATH=/home/app/.cache/ms-playwright
RUN PYTHONPATH="/home/app/.local/lib/python${PYTHON_VERSION}/site-packages" \
    python -m playwright install --with-deps chromium \
    && chown -R app:app /home/app/.cache

COPY --chown=app:app function/   .

FROM build AS ship
WORKDIR /home/app/

USER app

# Set up of-watchdog for HTTP mode
ENV fprocess="python index.py"
ENV cgi_headers="true"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:5000"

HEALTHCHECK --interval=5s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]
```

The key addition compared to the standard template is the `playwright install --with-deps chromium` step:

- `--with-deps` installs both the Chromium binary and all required system libraries (`libglib`, `libnss3`, `libgbm`, etc.) via apt.
- The browser is installed to `/home/app/.cache/ms-playwright`, set by the `PLAYWRIGHT_BROWSERS_PATH` env var.
- Ownership of that directory is transferred to the `app` user so the browser is accessible at runtime without elevated privileges.

You will also need to copy the `index.py` and `requirements.txt` files from the `python3-http-debian` template into your function handler directory. Unlike `faas-cli new`, the `dockerfile` language does not scaffold these files automatically — they need to be present alongside the Dockerfile at build time:

```bash
faas-cli template store pull python3-http-debian
cp template/python3-http-debian/index.py playwright-scrape/
cp template/python3-http-debian/requirements.txt playwright-scrape/
```

Then add the handler and function requirements:

`playwright-scrape/function/handler.py` — use the handler code from the overview above.

`playwright-scrape/function/requirements.txt` — add the `playwright` package so it is installed into the function's Python environment at build time:

```
playwright
```

### Deploy and invoke

Build, push and deploy the function:

```bash
faas-cli up \
 --filter playwright-scrape \
 --tag digest
```

Get the title of a webpage:

```bash
echo '{"uri": "https://docs.openfaas.com"}' | \
  faas-cli invoke playwright-scrape \
  --header "Content-Type: application/json"
```

```json
{"title": "Introduction - OpenFaaS"}
```

### Take a screenshot

To return a screenshot as a PNG, update the handler to capture the page and return the binary data:

```python
import json
from playwright.sync_api import sync_playwright

def handle(event, context):
    body = json.loads(event.body)
    uri = body.get("uri", "https://www.openfaas.com")

    with sync_playwright() as p:
        browser = p.chromium.launch(
            args=["--no-sandbox", "--disable-setuid-sandbox"]
        )
        try:
            page = browser.new_page()
            page.goto(uri, timeout=30000)
            screenshot = page.screenshot(full_page=True)
        finally:
            browser.close()

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "image/png"},
        "body": screenshot
    }
```

Invoke and save the screenshot:

```bash
echo '{"uri": "https://docs.openfaas.com"}' | \
  faas-cli invoke playwright-scrape \
  --header "Content-Type: application/json" > screenshot.png
```

The screenshot could also be uploaded directly to S3 from within the function instead of returning it to the caller. See [Access S3 object storage with boto3](s3-boto3.md) for how to set up an S3 client.

### Hardening

Each browser instance is memory-intensive. Set `max_inflight: "1"` to allow only one concurrent request per container replica, and add a memory limit to cap resource usage. Chromium requires at least 512Mi at idle — use 1Gi for production workloads, especially when taking full-page screenshots:

```yaml
functions:
  playwright-scrape:
    lang: dockerfile
    handler: ./playwright-scrape
    image: ttl.sh/openfaas-examples/playwright-scrape:latest
    environment:
      max_inflight: "1"
      PLAYWRIGHT_BROWSERS_PATH: /home/app/.cache/ms-playwright
    limits:
      memory: 1Gi
```

See also: [Web scraping that just works with OpenFaaS with Puppeteer](https://www.openfaas.com/blog/puppeteer-scraping/) and [Generate PDFs at scale on Kubernetes](https://www.openfaas.com/blog/pdf-generation-at-scale-on-kubernetes/) for patterns on scaling headless browsers with OpenFaaS.
