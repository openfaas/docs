# CI/CD with Bitbucket Pipelines

You can use Bitbucket Pipelines to build and publish container images for your OpenFaaS functions. As a final step, you may also wish to deploy the new images to the OpenFaaS gateway via the OpenFaaS CLI.

## Build and deploy

This pipeline builds and pushes function images, then deploys them to the OpenFaaS gateway. The build and deploy are defined as separate steps so they can be tracked independently.

Pre-requisites:

* A container registry accessible from the pipeline (e.g. Docker Hub, [Bitbucket Packages](https://support.atlassian.com/bitbucket-cloud/docs/getting-started-with-packages/), AWS ECR, or another private registry).
* The OpenFaaS gateway must be accessible from the Bitbucket pipeline runner.

Add the following repository variables under **Repository settings > Pipelines > Repository variables**:

| Variable | Description | Secured |
|---|---|---|
| `DOCKER_USERNAME` | Username for the container registry e.g. Docker Hub username | No |
| `DOCKER_PASSWORD` | Password or access token for the container registry | Yes |
| `OPENFAAS_URL` | URL of the OpenFaaS gateway e.g. `https://gw.example.com` | No |
| `OPENFAAS_PASSWORD` | Password for the OpenFaaS gateway | Yes |

!!! info "OpenFaaS for Enterprises"
    If you are using OpenFaaS for Enterprises, we recommend using [Web Identity Federation](/openfaas-pro/iam/bitbucket-federation/) using the OIDC token provided by Bitbucket Pipelines instead of sharing the admin password with your CI system. This avoids long-lived credentials and lets you scope access to specific namespaces, functions and actions.

See: [Variables and secrets](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)

### Create a function

If you don't already have a function, you can scaffold one with the `faas-cli`. The following example creates a Python function, but any supported language template can be used:

```bash
export OPENFAAS_PREFIX="docker.io/username"

faas-cli new --lang python3-http import-csv
```

This generates a `stack.yaml` and a handler directory.

Update `import-csv/handler.py` so it echos the request body:

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": event.body
    }
```

You can edit the image field in your `stack.yaml` to use [environment variable substitution](/reference/yaml/#yaml-environment-variable-substitution) so the registry and namespace aren't hard-coded.

The `configuration.templates` section declares the template repositories your functions depend on. In a CI pipeline, this means `faas-cli template pull stack` will automatically fetch the correct templates function templates.

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  import-csv:
    lang: python3-http
    handler: ./import-csv
    image: ${REGISTRY:-docker.io}/${NAMESPACE:-username}/import-csv:latest
configuration:
  templates:
    - name: python3-http
      source: https://github.com/openfaas/python-flask-template
```

### Create the pipeline

Create a `bitbucket-pipelines.yml` file in the root of your repository:

```yaml
image: atlassian/default-image:3

pipelines:
  branches:
    main:
      - step:
          name: Build and push
          services:
            - docker
          script:
            - curl -sLS https://cli.openfaas.com | sh

            # Login to the container registry
            - >-
              echo "$DOCKER_PASSWORD" |
              docker login --username "$DOCKER_USERNAME" --password-stdin

            # Build and push function images
            - faas-cli template pull stack
            - faas-cli build --tag=sha
            - faas-cli push --tag=sha

      - step:
          name: Deploy
          script:
            - curl -sLS https://cli.openfaas.com | sh

            # Login to the OpenFaaS gateway
            - >-
              echo "$OPENFAAS_PASSWORD" |
              faas-cli login --username admin --password-stdin

            # Deploy functions
            - faas-cli template pull stack
            - faas-cli deploy --tag=sha
```

The `--tag=sha` flag appends the Git commit SHA to the image tag so that each build produces a unique, traceable image. Both steps must use the same flag. Alternatives like `--tag=branch` and `--tag=describe` are also available, see [Image tagging](/cli/tags/). You can also template the image field in your stack.yaml using [environment variable substitution](/reference/yaml/#yaml-environment-variable-substitution) to tag images in whatever format you need.

Only the build step requires the Docker service since it builds container images. The deploy step only needs the `faas-cli` to communicate with the gateway's REST API.

If you are using a private registry, the OpenFaaS cluster must be able to pull images from it. See [Configure OpenFaaS to pull from a private registry](/reference/private-registries/).

## Optional: validate functions before merge

You can optionally add a build step that runs on pull requests to validate that functions build correctly before merging. This only builds the images without pushing them to a registry.

```yaml
image: atlassian/default-image:3

pipelines:
  pull-requests:
    '**':
      - step:
          name: Build functions
          services:
            - docker
          script:
            - curl -sLS https://cli.openfaas.com | sh
            - faas-cli template pull stack
            - faas-cli build
```

When multiple functions are available in the stack.yaml file you can add `--parallel` to speed up the build by building multiple functions at once.
