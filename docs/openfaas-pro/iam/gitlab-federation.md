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
    - Function:Read
    - Function:Admin
    - Secret:Read
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

## Create a GitLab CI/CD pipeline

To access the OpenFaaS gateway from a CI/CD pipeline you should adapt your job to:

- Obtain an id_token with the proper audience.
- Authenticate to OpenFaaS with the id_token using the faas-cli pro plugin.

Example `.gitlab-ci.yml` file:
```yaml
stages:
  - build

services:
  - docker:dind

cache:
  key: ${CI_COMMIT_REF_SLUG} # i.e. master
  paths:
  - ./faas-cli
  - ./template

build_job:
  stage: build
  image: docker:latest
  variables:
    OPENFAAS_URL: https://gw.example.com

  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - apk add --no-cache git curl
    - if [ -f "./faas-cli" ] ; then cp ./faas-cli /usr/local/bin/faas-cli || 0 ; fi
    - if [ ! -f "/usr/local/bin/faas-cli" ] ; then apk add --no-cache curl git && curl -sSL https://cli.openfaas.com | sh && chmod +x /usr/local/bin/faas-cli && cp /usr/local/bin/faas-cli ./faas-cli ; fi

    - faas-cli plugin get pro
    - faas-cli pro enable

    - faas-cli pro auth --token=$ID_TOKEN_1

    - faas-cli list -n dev
    - faas-cli ns
    - faas-cli store deploy -n dev printer --name p1

  id_tokens:
    ID_TOKEN_1:
      aud: https://gw.example.com
```

Within your GitLab job, you must obtain an id_token with the proper audience `aud` field set with the address of your OpenFaaS gateway.

Next the faas-cli pro plugin can be used to authenticate to the OpenFaaS gateway using the id_token. It will exchange the token for an OpenFaaS token and save it. The faas-cli can then be used to talk to the gateway.
