# CI/CD with GitLab

CI/CD can be achieved in a number of ways. This page outlines how to create a simple pipeline for automatically deploying new versions of your function every time there is a `git push` event on the repo.


## Build with `.gitlab-ci.yml`

To achieve CI/CD with GitLab you can create a file named `.gitlab-ci.yml` with the following contents:

```yaml
stages:
  - build

# Cache the templates and build-context to speed things up
cache:
  key: ${CI_COMMIT_REF_SLUG} # i.e. master
  paths:
  - ./faas-cli
  - ./template

# Build the whole stack using only the faas-cli
docker-build:
  stage: build
  image: docker:dind
  script:
    - apk add --no-cache git
    - if [ -f "./faas-cli" ] ; then cp ./faas-cli /usr/local/bin/faas-cli || 0 ; fi
    - if [ ! -f "/usr/local/bin/faas-cli" ] ; then apk add --no-cache curl git && curl -sSL cli.openfaas.com | sh && chmod +x /usr/local/bin/faas-cli && /usr/local/bin/faas-cli template pull && cp /usr/local/bin/faas-cli ./faas-cli ; fi

    # Build Docker image
    - /usr/local/bin/faas-cli build --tag=sha --parallel=2

    # Login & Push Docker image to private repo
    - echo -n "$CI_DOCKER_LOGIN" | docker login --username admin --password-stdin ae-reg.team-serverless.xyz
    - /usr/local/bin/faas-cli push --tag=sha
    - echo -n "$CI_OPENFAAS_PASSWORD" | /usr/local/bin/faas-cli login --username admin --password-stdin

    # Deploy function from private repo
    - /usr/local/bin/faas-cli deploy --tag=sha --send-registry-auth

  only:
    - master
```

> Note: You must also change the value of `ae-reg.team-serverless.xyz` to your own registry.

For Kubernetes you do not need the flag of `--send-registry-auth` for the `faas-cli deploy` command. 

Save this file in the git repo root of each set of functions you want to build and deploy.

## Build functions for your team or whole instance

It may be tiresome to maintain individual files for each repo and project.

See also: [OpenFaaS Cloud](../../openfaas-cloud/intro.md) which supports both GitHub and GitLab.