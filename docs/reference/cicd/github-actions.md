# GitHub Actions for building OpenFaaS Functions

You can use GitHub Actions to build or publish container images for your OpenFaaS functions. As a final step, you may also wish to deploy the new images to the OpenFaaS gateway via the OpenFaaS CLI.

If you'd like to deploy the function, check out a more comprehensive example of how to log in and deploy in [Serverless For Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else).

## A build to validate functions before merge

```yaml
name: build

on:
  push:
    branches:
      - '*'
  pull_request:
    branches:
      - '*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
        with:
          fetch-depth: 1
      - name: Get faas-cli
        uses: alexellis/arkade-get@master
        with:
          faas-cli: latest
      - name: Pull custom templates from stack.yml
        run: faas-cli template pull stack
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Get Repo Owner
        id: get_repo_owner
        run: >
          echo "repo_owner=$(echo ${{ github.repository_owner }} | tr '[:upper:]' '[:lower:]')" >> $GITHUB_OUTPUT
      - name: Build functions
        run: >
          OWNER="${{ steps.get_repo_owner.outputs.repo_owner }}" 
          TAG="latest"
          SERVER="ghcr.io"
          faas-cli build
          --build-arg GO111MODULE=on
          --filter release-promoter
```

The `--filter release-promoter` line can be removed to build all functions in the stack.yml.

When multiple functions are available in the stack.yaml file you can add `--parallel` to speed up the build by building multiple functions at once.

## Publish multiple functions

We will deploy alexellis' repository called [alexellis/autoscaling-functions](https://github.com/alexellis/autoscaling-functions). It contains multiple functions which can be deployed as a group.

* The "Setup QEMU" and "Set up Docker Buildx" steps configure the builder to produce a multi-arch image.
* The "OWNER" variable means this action can be run on any organisation without having to hard-code a username for GHCR.
* Only the bcrypt function is being built with the `--filter` command added, remove it to build all functions in the stack.yml.
* `--platforms linux/amd64,linux/arm64,linux/arm/v7` will build for regular Intel/AMD machines, 64-bit Arm and 32-bit Arm i.e. Raspberry Pi, most users can reduce this list to just "linux/amd64" for a speed improvement

If you're using [actuated.dev](https://actuated.dev) for fast, secure self-hosted runners, then you should `runs-on:` to the amount of vCPU and RAM you want i.e. `runs-on: actuated-4cpu-16gb`.

```yaml
name: publish

on:
  push:
    tags:
      - '*'

permissions:
  actions: read
  checks: write
  contents: read
  packages: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
        with:
          fetch-depth: 1
      - name: Get faas-cli
        uses: alexellis/arkade-get@master
        with:
          faas-cli: latest
      - name: Pull custom templates from stack.yml
        run: faas-cli template pull stack
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Get TAG
        id: get_tag
        run: echo "TAG=latest-dev" >> $GITHUB_OUTPUT
      - name: Get Repo Owner
        id: get_repo_owner
        run: >
          echo "repo_owner=$(echo ${{ github.repository_owner }} | tr '[:upper:]' '[:lower:]')" >> $GITHUB_OUTPUT
      - name: Docker Login
        run: > 
          echo ${{secrets.GITHUB_TOKEN}} | 
          docker login ghcr.io --username 
          ${{ steps.get_repo_owner.outputs.repo_owner }} 
          --password-stdin
      - name: Publish functions
        run: >
          OWNER="${{ steps.get_repo_owner.outputs.repo_owner }}" 
          TAG="latest"
          SERVER="ghcr.io"
          faas-cli publish
          --extra-tag ${{ github.sha }}
          --extra-tag ${{ steps.get_tag.outputs.TAG }}
          --build-arg GO111MODULE=on
          --platforms linux/amd64,linux/arm64
          --filter release-promoter
```

The Publish functions step uses [environment substitution](/reference/yaml/#yaml-environment-variable-substitution) to set the owner and tag for the image.
The tag for the image is published as "latest", along with an extra tag of the SHA from the commit from GitHub. The Owner portion of the image is set via a dynamic variable too, so that the repository can be forked and run under a different user account.

`ghcr.io/${OWNER:-alexellis}/bcrypt:${TAG:-latest}`

Any other text or variables can also be substituted in this way, making it easy to re-use the same workflow across multiple repositories, user accounts, or branches.

### Deploy functions from CI

To deploy functions, you can use the `faas-cli` in a `run` step.

First run `faas-cli login` with the gateway's public URL, then run `faas-cli deploy`.

If you do not have a public URL for your OpenFaaS gateway, or are running within a private VPC, there are a couple of options:

1. Don't expose anything, but establish a private link as required - [Deploy to a private cluster from GitHub Actions without exposing it to the Internet](https://inlets.dev/blog/2021/12/06/private-deploy-from-github-actions.html)
2. Keep the VPC private, but expose only the OpenFaaS gateway - [Expose your local OpenFaaS functions to the Internet](https://inlets.dev/blog/2020/10/15/openfaas-public-endpoints.html)

By default, all functions in stack.yaml will be deployed, or you can add the `--filter` flag to a named function.

If you are using OpenFaaS Standard, then you will need to create a password for the repository or organisation with the shared admin password for the gateway.

If you're using IAM for OpenFaaS, then you can use Web Identity Federation, and a short-lived OIDC token from GitHub, see also: [GitHub Actions - Web Identity Federation](/openfaas-pro/iam/github-actions-federation/). This option is much more secure, and means that not only can the job be scoped to only what it needs to change - a single namespace, or a subset of objects, but it also means that no long-lived credentials are shared with the CI system.

#### Dashboard integration

There are various fields on the dashboard such as the Commit SHA, the repository owner and the repository URL. When metadata is populated via labels and annotations on your function, then these become links, and will show for your team.

See the notes here for a full list of metadata options: [OpenFaaS Dashboard](/openfaas-pro/dashboard/)

## Further reading

See our reference manual for OpenFaaS for recipes and examples:

* [Serverless For Everyone Else](http://store.openfaas.com/l/serverless-for-everyone-else)

