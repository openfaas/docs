# Bitbucket - Web Identity Federation

In this guide, you'll learn how to deploy from Bitbucket Pipelines using OpenFaaS's IAM support and Web Identity Federation.

You'll need to create YAML files for an Issuer, a Policy and a Role. These need to be applied through kubectl, Helm or a GitOps tool.

Your build will need to be adapted in order to receive an *id_token* from Bitbucket, which will be exchanged for an OpenFaaS access token.

## Pre-requisites

### OIDC identity provider

You need to retrieve your workspace's OIDC identity provider URL and audience from Bitbucket:

1. Navigate to your repository in Bitbucket Cloud.
2. Select **Repository settings** on the left navigation sidebar.
3. Select **OpenID Connect** under **Pipelines** on the left sidebar.
4. Copy the **Identity provider URL** and **Audience** values.

![Bitbucket OpenID Connect settings page](/images/iam/bitbucket-oidc-settings.png)

You'll need both values when configuring the `JwtIssuer` resource below.

See also: [Integrate Pipelines with resource servers using OIDC](https://support.atlassian.com/bitbucket-cloud/docs/integrate-pipelines-with-resource-servers-using-oidc/)

### faas-cli pro license

The `faas-cli pro` plugin requires a valid CLI license. The plugin looks for the license in the `OPENFAAS_CLI_LICENSE` environment variable.

Add `OPENFAAS_CLI_LICENSE` as a repository variable under **Repository settings > Pipelines > Repository variables**, or as a workspace variable under **Workspace settings > Pipelines > Workspace variables**. Mark it as **Secured** so its value is masked in build logs. Workspace and repository variables are automatically available as environment variables in all pipeline steps.

See: [Variables and secrets](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)

> If you don't have a CLI license yet, contact your account representative at OpenFaaS to receive one.

## Define an Issuer for Bitbucket

First define a new JwtIssuer resource, setting the `iss` field to the Identity provider URL retrieved from your workspace's OIDC settings. Set the `aud` field to the URL of your OpenFaaS gateway.

```yaml
apiVersion: iam.openfaas.com/v1
kind: JwtIssuer
metadata:
  name: bitbucket.com
  namespace: openfaas
spec:
  iss: https://api.bitbucket.org/2.0/workspaces/WORKSPACE/pipelines-config/identity/oidc
  aud:
  - https://gw.example.com
  tokenExpiry: 30m
```

Replace `WORKSPACE` with the name of your Bitbucket workspace.

The `tokenExpiry` field controls how long the OpenFaaS access token is valid. A short expiry such as `30m` is recommended to reduce the window in which a leaked token could be used, but it can be increased for longer-running CI jobs.

The `aud` field must match the custom audience configured in your Bitbucket pipeline. By adding the gateway URL as a [custom audience](#custom-audience) in the pipeline step, it is included in the token's `aud` claim so that OpenFaaS can validate the token.

If you have repositories across multiple workspaces that need to access OpenFaaS, you will need to create a separate issuer for each workspace.

## Create a Policy

Next, define a Policy with the privileges required to perform the desired actions. The following examples show two common use cases:

- A deployment role for any function within the `etl` namespace
- An invocation-only role for a single function in the `etl` namespace

The `etl-deployer` policy allows listing and deploying functions in the `etl` namespace. It grants permission to list namespaces, list functions, and create or update any function within `etl`:

```yaml
apiVersion: iam.openfaas.com/v1
kind: Policy
metadata:
  name: etl-deployer
  namespace: openfaas
spec:
  statement:
  - sid: 1-rw-etl
    action:
    - Function:List
    - Function:Create
    - Function:Update
    - Namespace:List
    effect: Allow
    resource: ["etl:*"]
```

For a pipeline that only needs to call a function, the `etl-invoker` policy restricts access to invoking a single function by name:

```yaml
apiVersion: iam.openfaas.com/v1
kind: Policy
metadata:
  name: etl-invoker
  namespace: openfaas
spec:
  statement:
  - sid: 1-invoke-etl
    action:
    - Function:Invoke
    effect: Allow
    resource: ["etl:my-function"]
```

## Bind a Policy to a Role

Next, you need to bind the Policy to a Role.

The Role uses conditions to match claims from the Bitbucket OIDC token. Here is an overview of the claims available within a Bitbucket Pipelines `id_token`:

```json
{
  "sub": "{repositoryUuid}:{stepUuid}",
  "stepUuid": "{xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx}",
  "iss": "https://api.bitbucket.org/2.0/workspaces/WORKSPACE/pipelines-config/identity/oidc",
  "branchName": "xxxxxxxxx",
  "workspaceUuid": "{xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx}",
  "pipelineRunUuid": "{xxxxx-xxxxx-xxxx-xxxx-xxxxxxxxxxx}",
  "aud": [
    "ari:cloud:bitbucket::workspace/WORKSPACE",
    "https://gw.example.com"
  ],
  "repositoryUuid": "{xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx}",
  "exp": "xxxxxxxxxx",
  "iat": "xxxxxxxxxx",
  "pipelineUuid": "{xxxxx-xxxxx-xxxx-xxxx-xxxxxxxxxxx}"
}
```

You can view the example payload for your repository on the **OpenID Connect** page under **Repository settings > Pipelines** in Bitbucket Cloud.

When a [custom audience](#custom-audience) is configured in the pipeline, it will appear alongside the default workspace audience in the `aud` array.

```yaml
apiVersion: iam.openfaas.com/v1
kind: Role
metadata:
  name: etl-ci-deployer
  namespace: openfaas
spec:
  policy:
  - etl-deployer
  condition:
    StringEqual:
      jwt:iss: ["https://api.bitbucket.org/2.0/workspaces/WORKSPACE/pipelines-config/identity/oidc"]
      jwt:repositoryUuid: ["{REPO_UUID}"]
      jwt:branchName: ["main"]
```

The example above restricts deployments to a specific repository and the `main` branch. You could restrict this further by matching additional claims such as `workspaceUuid` or `pipelineUuid`.

Finally, you need to apply all of the above objects.

## Create a Bitbucket Pipeline

To access the OpenFaaS gateway from a Bitbucket Pipeline you should adapt your pipeline to:

- Obtain the id_token via the `BITBUCKET_STEP_OIDC_TOKEN` environment variable.
- Authenticate to OpenFaaS with the id_token using the faas-cli pro plugin.

### Custom audience

By default, the Bitbucket OIDC token only includes the workspace audience (e.g. `ari:cloud:bitbucket::workspace/WORKSPACE_UUID`). We recommend adding the OpenFaaS gateway URL as a custom audience. This is done by adding an `oidc` block with an `audiences` list to your pipeline step. The custom audience is appended to the default workspace audience in the token's `aud` claim.

```yaml
oidc:
  audiences:
    - https://gw.example.com
```

When setting a custom audience in the pipeline, the `aud` field in the `JwtIssuer` resource must be set to the same gateway URL so that OpenFaaS can validate the token.

You can set custom audiences at the step level or globally using `options`. See the [Bitbucket documentation](https://support.atlassian.com/bitbucket-cloud/docs/integrate-pipelines-with-resource-servers-using-oidc/) for more details.

### Example pipeline

This is an example `bitbucket-pipelines.yml` file:

```yaml
image: atlassian/default-image:3

pipelines:
  default:
    - step:
        name: Deploy to OpenFaaS
        services:
          - docker
        oidc:
          audiences:
            - https://gw.example.com
        script:
          - export OPENFAAS_URL="https://gw.example.com"

          - mkdir -p $HOME/bin
          - export PATH=$PATH:$HOME/bin
          - curl -sLS https://cli.openfaas.com | sh

          - faas-cli plugin get pro

          - faas-cli pro auth --token=$BITBUCKET_STEP_OIDC_TOKEN

          - faas-cli list -n etl
          - faas-cli ns
          - faas-cli store deploy -n etl printer --name p1
```

The `oidc.audiences` field adds the OpenFaaS gateway URL as a custom audience to the OIDC token. Bitbucket makes the token available through the `BITBUCKET_STEP_OIDC_TOKEN` environment variable when `oidc` is configured on the step.

The faas-cli pro plugin exchanges the Bitbucket OIDC token for an OpenFaaS access token and saves it. After that, faas-cli can be used to interact with the gateway.
