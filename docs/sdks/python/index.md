The [OpenFaaS Python SDK](https://github.com/openfaas/python-sdk) provides typed access to the full [OpenFaaS REST API](/reference/rest-api/) from Python, including the [Function Builder API](/openfaas-pro/builder/).

## Installation

```bash
pip install git+https://github.com/openfaas/python-sdk.git
```

## Features

- Manage functions, namespaces, and secrets
- Invoke functions synchronously and asynchronously
- Stream function logs
- Build and push function images from source via the Function Builder API
- Basic auth and [OpenFaaS IAM](/openfaas-pro/iam/overview/) support

## Examples

* [From source to function](examples/source-to-function.md)
