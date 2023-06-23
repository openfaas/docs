# GitHub Actions - Web Identity Federation

In this guide, you'll learn how to deploy from GitHub Actions CI/CD using OpenFaaS's IAM support and Web Identity Federation. 

You'll need to create YAML files for an Issuer, a Policy and a Role. These need to be applied through kubectl, Helm or a GitOps tool.

Your build will need to be adapted in order to receive an id_token from GitHub, which will be exchanged for an OpenFaaS access token.

## Define an Issuer for GitHub Actions

First define a new JwtIssuer resource, setting the `aud` field to the URL of your OpenFaaS Gateway.

```yaml
apiVersion: iam.openfaas.com/v1
kind: JwtIssuer
metadata:
  name: token.actions.githubusercontent.com
  namespace: openfaas
spec:
  iss: https://token.actions.githubusercontent.com
  aud:
  - https://gw.example.com
  tokenExpiry: 30m
```

> Issuer for https://token.actions.githubusercontent.com

## Create a Policy

Next, define a Policy with the least privileges required to perform the desired actions.

```yaml
apiVersion: iam.openfaas.com/v1
kind: Policy
metadata:
  name: dev-rw
  namespace: openfaas
spec:
  statement:
  - sid: 1-rw-dev
    action:
    - Function:List
    - Function:Create
    - Namespace:List
    effect: Allow
    resource: ["dev:*"]
```

## Bind a Policy to a Role

Next, you need to bind the Policy to a Role.

There are around a dozen different fields available within the GitHub Actions `id_token`:

```json
{
  "actor": "aidansteele",
  "aud": "https://github.com/aidansteele/aws-federation-github-actions",
  "base_ref": "",
  "event_name": "push",
  "exp": 1631672856,
  "head_ref": "",
  "iat": 1631672556,
  "iss": "https://token.actions.githubusercontent.com",
  "job_workflow_ref": "aidansteele/aws-federation-github-actions/.github/workflows/test.yml@refs/heads/main",
  "jti": "8ea8373e-0f9d-489d-a480-ac37deexample",
  "nbf": 1631671956,
  "ref": "refs/heads/main",
  "ref_type": "branch",
  "repository": "aidansteele/aws-federation-github-actions",
  "repository_owner": "aidansteele",
  "run_attempt": "1",
  "run_id": "1235992580",
  "run_number": "5",
  "sha": "bf96275471e83ff04ce5c8eb515c04a75d43f854",
  "sub": "repo:aidansteele/aws-federation-github-actions:ref:refs/heads/main",
  "workflow": "CI"
}
```

> Example from: [Deploy without credentials with GitHub Actions and OIDC](https://blog.alexellis.io/deploy-without-credentials-using-oidc-and-github-actions/)

```yaml
apiVersion: iam.openfaas.com/v1
kind: Role
metadata:
  name: dev-actions-deployer
  namespace: openfaas
spec:
  policy:
  - dev-rw
  condition:
    StringEqual:
      jwt:iss: ["https://token.actions.githubusercontent.com"]
      jwt:repository_owner: ["openfaas"]
    StringLike:
      jwt:ref: ["refs/heads/*"]
```

The example must match the issuer and organisation name of "openfaas", and can match any branch name.

You could restrict this further by looking at the "actor" for instance.

Finally, you need to apply all of the above objects.

## Create a GitHub Actions workflow

To access the OpenFaaS gateway from a workflow you should adapt the workflow to:

- Fetch a GitHub Actions id_token
- Authenticate to OpenFaaS with the id_token using the faas-cli pro plugin.

This is an example of a workflow file:
```yaml
name: federate

on:
  workflow_dispatch:
  push:
    branches:
    - '*'
jobs:
  auth:
    # Add "id-token" with the intended permissions.
    permissions:
      contents: 'read'
      id-token: 'write'

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
        with:
          fetch-depth: 1
      - name: Dump env
        run: env
      - name: Install faas-cli
        run: curl -sLS https://cli.openfaas.com | sudo sh          
      - name: Get token and use the CLI
        run: |
          export OPENFAAS_URL="https://gw.example.com"

          OIDC_TOKEN=$(curl -sLS "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=$OPENFAAS_URL" -H "User-Agent: actions/oidc-client" -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN")
          JWT=$(echo $OIDC_TOKEN | jq -j '.value')

          faas-cli plugin get pro
          faas-cli pro enable

          faas-cli pro auth --token=$JWT

          faas-cli list -n dev
          faas-cli ns
          faas-cli store deploy -n dev printer --name p1
```

First a GitHub Actions id_token is fetched. The workflow uses the faas-cli pro plugin to authenticate to the OpenFaaS gateway using the GitHub Actions id_token. It will exchange the token for an OpenFaaS token and save it. Next the faas-cli can be used to talk to the gateway.
