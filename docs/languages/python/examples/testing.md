Unit test your OpenFaaS Python functions using [pytest](https://docs.pytest.org/) and [tox](https://tox.wiki/). Tests run without deploying, giving fast feedback during development.

Use-cases:

* Validating request parsing and response formatting
* Testing business logic in isolation
* Running tests in CI before deploying

This example shows a small calculator function that validates input with `pydantic`, and a test suite that exercises it by mocking the `event` object.

## Overview

handler.py:

```python
import json
from pydantic import BaseModel, validator

class CalcRequest(BaseModel):
    op: str
    var1: float
    var2: float

    @validator("op")
    def op_must_be_valid(cls, v):
        if v not in ("+", "-", "*", "/"):
            raise ValueError(f"unsupported operation: {v}")
        return v

def handle(event, context):
    try:
        req = CalcRequest(**json.loads(event.body))
    except Exception as e:
        return {
            "statusCode": 400,
            "body": {"error": str(e)}
        }

    if req.op == "+":
        result = req.var1 + req.var2
    elif req.op == "-":
        result = req.var1 - req.var2
    elif req.op == "*":
        result = req.var1 * req.var2
    elif req.op == "/":
        if req.var2 == 0:
            return {
                "statusCode": 400,
                "body": {"error": "division by zero"}
            }
        result = req.var1 / req.var2

    return {
        "statusCode": 200,
        "body": {"value": result}
    }
```

handler_test.py:

```python
from . import handler as h
from types import SimpleNamespace

def make_event(body, method="POST"):
    return SimpleNamespace(
        body=body,
        headers={},
        path="/",
        method=method,
    )

class TestCalculator:
    def test_addition(self):
        event = make_event('{"op": "+", "var1": 1.0, "var2": 2.0}')
        resp = h.handle(event, {})
        assert resp["statusCode"] == 200
        assert resp["body"]["value"] == 3.0

    def test_subtraction(self):
        event = make_event('{"op": "-", "var1": 5.0, "var2": 3.0}')
        resp = h.handle(event, {})
        assert resp["statusCode"] == 200
        assert resp["body"]["value"] == 2.0

    def test_division_by_zero(self):
        event = make_event('{"op": "/", "var1": 1.0, "var2": 0}')
        resp = h.handle(event, {})
        assert resp["statusCode"] == 400

    def test_invalid_operator(self):
        event = make_event('{"op": "^", "var1": 1.0, "var2": 2.0}')
        resp = h.handle(event, {})
        assert resp["statusCode"] == 400
```

requirements.txt:

```
pydantic
```

## Step-by-step walkthrough

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-http
faas-cli new --lang python3-http calc \
  --prefix ttl.sh/openfaas-examples
```

Update `calc/handler.py` and `calc/requirements.txt` with the code from the overview above.

### Add a test file

The `python3-http` template scaffolds a `tox.ini` and a skeleton `handler_test.py` inside the function's handler directory. Replace the contents of `calc/handler_test.py` with the test code from the overview.

The `tox.ini` is preconfigured to discover and run `pytest` tests. You can inspect it with:

```bash
cat calc/tox.ini
```

### Build and run the tests

Testing is disabled by default in the `python3-http` template. Enable it by passing `--build-arg TEST_ENABLED=true` to `faas-cli build`:

```bash
faas-cli build \
  --filter calc \
  --tag digest \
  --build-arg TEST_ENABLED=true
```

During the build you should see output similar to:

```
handler_test.py ....                                                     [100%]

========================= 4 passed in 0.14s =========================
```

To enable testing permanently, add `TEST_ENABLED` to your stack.yml:

```yaml
functions:
  calc:
    lang: python3-http
    handler: ./calc
    image: ttl.sh/openfaas-examples/calc:latest
    build_args:
      TEST_ENABLED: "true"
```

### Deploy and invoke

Build, push and deploy the function:

```bash
faas-cli up \
  --filter calc \
  --tag digest
```

Invoke the calculator:

```bash
curl -s http://127.0.0.1:8080/function/calc \
  -d '{"op": "+", "var1": 1, "var2": 2}'
```
