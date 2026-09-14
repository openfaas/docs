OpenFaaS SDKs provide typed access to the [OpenFaaS REST API](/reference/rest-api/) and the [Function Builder API](/openfaas-pro/builder/) for managing and automating functions from your own code.

## Python SDK

The [OpenFaaS Python SDK](https://github.com/openfaas/python-sdk) provides access to the full OpenFaaS REST API and the Function Builder API from Python.

**Install:**

```bash
pip install git+https://github.com/openfaas/python-sdk.git
```

**Features:**

- Manage functions, namespaces, and secrets
- Invoke functions synchronously and asynchronously
- Stream function logs
- Build and push function images from source via the Function Builder API
- Basic auth and [OpenFaaS IAM](/openfaas-pro/iam/overview/) support

**Examples:**

- [From source to function](python/examples/source-to-function.md)

## Go SDK

The [OpenFaaS Go SDK](https://github.com/openfaas/go-sdk) provides access to the full OpenFaaS REST API and the Function Builder API from Go.

**Install:**

```bash
go get github.com/openfaas/go-sdk
```

**Features:**

- Manage functions, namespaces, and secrets
- Invoke functions synchronously and asynchronously
- Stream function logs
- Build and push function images from source via the Function Builder API
- Basic auth and [OpenFaaS IAM](/openfaas-pro/iam/overview/) support

**Examples:**

- [From source to function](go/examples/source-to-function.md)
