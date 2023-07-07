# GitLab - Web Identity Federation

In this guide, you'll learn how to deploy from GitLab CI/CD using OpenFaaS's IAM support and Web Identity Federation. 

You'll need to create YAML files for an Issuer, a Policy and a Role. These need to be applied through kubectl, Helm or a GitOps tool.

Your build will need to be adapted in order to receive an id_token from GitLab, which will be exchanged for an OpenFaaS access token.

## Define an Issuer for GitLab.com

First define a new JwtIssuer resource, setting the `aud` field to the URL of your OpenFaaS Gateway.

```yaml
apiVersion: iam.openfaas.com/v1
kind: JwtIssuer
metadata:
  name: gitlab.com
  namespace: openfaas
spec:
  iss: https://gitlab.com
  aud:
  - https://gw.example.com
  tokenExpiry: 30m
```

> Issuer for https://gitlab.com

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

There are around a dozen different fields available within GitLab's `id_token`, you can view a complete list at: [GitLab OIDC: Shared information](https://docs.gitlab.com/ee/integration/openid_connect_provider.html#shared-information)

```yaml
apiVersion: iam.openfaas.com/v1
kind: Role
metadata:
  name: gitlab-dev-actions-deployer
  namespace: openfaas
spec:
  policy:
  - dev-rw
  condition:
    StringEqual:
      jwt:iss: ["https://gitlab.com"]
      jwt:user_login: ["alexellis"]
    StringLike:
      jwt:project_path: ["consortia/*"]
```

The example must match the GitLab issuer, for the login of "alexellis", with any project within the "consortia" group.

Within your GitLab job, you must obtain an id_token with the proper audience `aud` field set with the address of your OpenFaaS gateway:

```yaml
  id_tokens:
    ID_TOKEN_1:
      aud: https://gw.example.com
```

See an example repository and `.gitlab-ci.yml` file on GitLab [gitlab.com/consortia/deploy-fn](https://gitlab.com/consortia/deploy-fn/-/blob/main/.gitlab-ci.yml)
