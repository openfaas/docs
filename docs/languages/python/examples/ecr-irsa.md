Call AWS services from a function using ambient credentials instead of static access keys. With [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html), the function's pod is automatically assigned temporary credentials via a Kubernetes Service Account mapped to an IAM role.

Use-cases:

* Accessing any AWS service (S3, DynamoDB, SQS, ECR, etc.) without static keys
* Meeting security policies that prohibit long-lived credentials
* Simplifying secret rotation by relying on short-lived tokens

This example creates and queries ECR repositories using `boto3`, but the same approach works for any AWS service. It requires OpenFaaS to be deployed on [AWS EKS](https://aws.amazon.com/eks/) with IRSA enabled. See [Creating an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) for setup, or [Manage AWS Resources from OpenFaaS Functions With IRSA](https://www.openfaas.com/blog/irsa-functions/) for an end-to-end walkthrough.

## Overview

handler.py:

```python
import os
import json
import boto3

ecrClient = None

def initECR():
    session = boto3.Session(
        region_name=os.getenv('AWS_REGION'),
    )
    return session.client('ecr')

def handle(event, context):
    global ecrClient

    if ecrClient is None:
        ecrClient = initECR()

    if event.method != 'POST':
        return {
            "statusCode": 405,
            "body": "Method not allowed"
        }

    body = json.loads(event.body)
    name = body.get('name')

    if not name:
        return {
            "statusCode": 400,
            "body": "Missing in body: name"
        }

    # Check if the repository already exists
    try:
        ecrClient.describe_repositories(repositoryNames=[name])
        return {
            "statusCode": 200,
            "body": json.dumps({"message": "Repository already exists"})
        }
    except ecrClient.exceptions.RepositoryNotFoundException:
        pass

    # Create the repository
    response = ecrClient.create_repository(
        repositoryName=name,
        imageTagMutability='MUTABLE',
        encryptionConfiguration={
            'encryptionType': 'AES256',
        },
        imageScanningConfiguration={
            'scanOnPush': False,
        },
    )

    return {
        "statusCode": 201,
        "body": json.dumps({
            "arn": response['repository']['repositoryArn']
        })
    }
```

requirements.txt:

```
boto3
```

stack.yaml:

```yaml
functions:
  ecr-create-repo:
    lang: python3-http-debian
    handler: ./ecr-create-repo
    image: ttl.sh/openfaas-examples/ecr-create-repo:latest
    annotations:
      com.openfaas.serviceaccount: openfaas-create-ecr-repo
    environment:
      AWS_REGION: eu-west-1
```

No secrets are needed. The `com.openfaas.serviceaccount` annotation tells OpenFaaS which Kubernetes Service Account to attach to the function's pod. EKS then mounts a short-lived token for that service account, and the AWS SDK picks up the credentials automatically — no access keys to store or rotate.

The `AWS_REGION` environment variable is required by the SDK to know which region to connect to.

## Step-by-step walkthrough

### Create an IAM Policy

Create a policy that grants the permissions your function needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:CreateRepository",
        "ecr:DeleteRepository",
        "ecr:DescribeRepositories"
      ],
      "Resource": "*"
    }
  ]
}
```

Save the above to `ecr-policy.json` and create the policy:

```bash
aws iam create-policy \
  --policy-name ecr-create-query-repository \
  --policy-document file://ecr-policy.json
```

Note the ARN from the output, e.g. `arn:aws:iam::ACCOUNT_NUMBER:policy/ecr-create-query-repository`.

### Create an IAM Role and Kubernetes Service Account

Use `eksctl` to create a Kubernetes Service Account in the `openfaas-fn` namespace that is linked to an IAM role with the policy attached:

```bash
export ARN=arn:aws:iam::ACCOUNT_NUMBER:policy/ecr-create-query-repository

eksctl create iamserviceaccount \
  --name openfaas-create-ecr-repo \
  --namespace openfaas-fn \
  --cluster <cluster-name> \
  --role-name ecr-create-query-repository \
  --attach-policy-arn $ARN \
  --region eu-west-1 \
  --approve
```

This can also be done manually by creating the IAM Role in AWS, followed by a Kubernetes Service Account annotated with `eks.amazonaws.com/role-arn`.

### Create the function

Pull the template and scaffold a new function:

```bash
faas-cli template store pull python3-http-debian
faas-cli new --lang python3-http-debian ecr-create-repo \
  --prefix ttl.sh/openfaas-examples
```

Update `ecr-create-repo/handler.py` and `ecr-create-repo/requirements.txt` with the code from the overview above.

### Deploy and invoke

Build, push and deploy the function with `faas-cli up`:

```bash
faas-cli up \
 --filter ecr-create-repo \
 --tag digest
```

Create a new ECR repository by invoking the function:

```bash
curl -X POST http://127.0.0.1:8080/function/ecr-create-repo \
  -H "Content-Type: application/json" \
  -d '{"name":"tenant1/fn1"}'
```

The response contains the ARN of the newly created repository:

```json
{"arn": "arn:aws:ecr:eu-west-1:ACCOUNT_NUMBER:repository/tenant1/fn1"}
```
