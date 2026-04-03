Use the `boto3` SDK to interact with [AWS S3](https://aws.amazon.com/s3/) or any S3-compatible object storage from an OpenFaaS function. Credentials are stored as [OpenFaaS secrets](/reference/secrets/).

Use-cases:

* Storing uploaded files or generated reports in S3
* Listing and serving assets from a bucket
* Backing up data or exporting results to object storage

This example lists and uploads objects in an AWS S3 bucket. The code also works with self-hosted S3-compatible backends such as [SeaweedFS](https://github.com/seaweedfs/seaweedfs), [RustFS](https://github.com/rustfs/rustfs), [Garage](https://garagehq.deuxfleurs.fr/), and [Ceph](https://docs.ceph.com/en/latest/radosgw/).

## Overview

handler.py:

```python
import os
import json
import boto3

# Initialise the S3 client once and reuse it across invocations
# to avoid re-reading secrets and creating a new session on every request.
s3Client = None

def initS3():
    with open('/var/openfaas/secrets/s3-key', 'r') as s:
        s3Key = s.read().strip()
    with open('/var/openfaas/secrets/s3-secret', 'r') as s:
        s3Secret = s.read().strip()

    session = boto3.Session(
        aws_access_key_id=s3Key,
        aws_secret_access_key=s3Secret,
    )

    return session.client('s3')

def handle(event, context):
    global s3Client

    if s3Client is None:
        s3Client = initS3()

    bucketName = os.getenv('s3_bucket')

    # GET — list all object keys in the bucket
    if event.method == 'GET':
        response = s3Client.list_objects_v2(Bucket=bucketName)
        keys = [obj['Key'] for obj in response.get('Contents', [])]

        return {
            "statusCode": 200,
            "body": json.dumps(keys)
        }

    # POST — upload the request body as an object
    elif event.method == 'POST':
        key = event.query.get('key', 'upload.txt')
        s3Client.put_object(Bucket=bucketName, Key=key, Body=event.body)

        return {
            "statusCode": 201,
            "body": "Uploaded to {}".format(key)
        }

    return {
        "statusCode": 405,
        "body": "Method not allowed"
    }
```

requirements.txt:

```
boto3
```

stack.yaml:

```yaml
functions:
  s3-example:
    lang: python3-http
    handler: ./s3-example
    image: ttl.sh/openfaas-examples/s3-example:latest
    environment:
      s3_bucket: my-bucket
    secrets:
    - s3-key
    - s3-secret
```

- The S3 client is initialised once on first invocation and reused for subsequent requests, avoiding the overhead of re-reading secrets and creating a new session on every call.
- A GET request lists all object keys in the bucket.
- A POST request uploads the request body as an object; the key is taken from the `?key=` query parameter.

To use a self-hosted S3-compatible backend, pass a custom `endpoint_url` when creating the client:

```python
    return session.client('s3', endpoint_url='https://s3.my-storage.example.com')
```

## Step-by-step walkthrough

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-http
faas-cli new --lang python3-http s3-example \
  --prefix ttl.sh/openfaas-examples
```

The example uses the public [ttl.sh](https://ttl.sh) registry — replace the prefix with your own registry for production use.

Update `s3-example/handler.py` and `s3-example/requirements.txt` with the code from the overview above.

### Create secrets for S3 credentials

Store your S3 access key and secret key as OpenFaaS secrets. This keeps credentials out of environment variables and the function's container image.

Save your access key ID to `s3-key.txt` and your secret access key to `s3-secret.txt`, then run:

```bash
faas-cli secret create s3-key --from-file s3-key.txt
faas-cli secret create s3-secret --from-file s3-secret.txt
```

At runtime, the secrets are mounted as files under `/var/openfaas/secrets/` inside the function container.

### Deploy and invoke

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter s3-example \
 --tag digest
```

Upload a file to the bucket with a POST request. The `?key=` query parameter sets the object key — this maps to `event.query.get('key', 'upload.txt')` in the handler:

```bash
curl -X POST http://127.0.0.1:8080/function/s3-example?key=hello.txt \
  --data "Hello from OpenFaaS"
```

List all objects in the bucket with a GET request:

```bash
curl http://127.0.0.1:8080/function/s3-example
```
