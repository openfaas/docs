# YAML format reference

This page covers the OpenFaaS YAML stack file used to configure functions.

The YAML file can hold one to many functions separated by separate entries.

Example:

```bash
$ faas-cli new --lang go fn1
$ faas-cli new --lang go fn2 --append=fn1.yml
```

Produces:

```YAML
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  fn1:
    lang: go
    handler: ./fn1
    image: fn1:latest
  fn2:
    lang: go
    handler: ./fn2
    image: fn2:latest
```

### Provider

The only valid value for provider `name` is `openfaas`.

#### Gateway

The gateway URL can be hard-coded into the YAML file or overriden at deployment time with the --gateway flag or OPENFAAS_URL env-var.

### Functions

The `functions` element holds a map of functions, by default all functions are acted on with CLI verbs, but you can filter them with `--filter` or `--regex`.

#### Function Name

The function Name is specified by a key in the functions map, i.e. `fn1` in the above example. Function name must be unique within a `stack.yml` file.

Valid function names follow ietf [rfc1035](https://tools.ietf.org/html/rfc1035) which is also used for DNS sub-domains.

> (DNS_LABEL): An alphanumeric (a-z, and 0-9) string, with a maximum length of 63 characters, with the '-' character allowed anywhere except the first or last character, suitable for use as a hostname or segment in a domain name.

#### Function: Language

The `lang` field refers to which template is going to be used to build the function. The templates are expected to be found in the ./template folder and will be pulled from GitHub if not present.

#### Function: Handler

The function `handler` field refers to a folder where the function's source code can be found, it must always be a folder and not a filename.

#### Function: Image

The `image` field refers to a Docker image reference, this could be on the Docker Hub, in your local Docker library or on another remote server.

#### Function: Skip build

The `skip_build` field controls whether the CLI will attempt to build the Docker image for the function.  When `true`, the build step is skipped and you should see a message printed to the terminal `Skipping build of: "function name"`.

This an optional boolean field, set to `false` by default.

#### Function: Build Options

The `build_options` field can be used to you to pass a list of [Docker build arguments](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg) to the build process.  When the language template supports it, this allows you to customize the build without modifying the underlying template.

For example, the [official python3 language template](https://github.com/openfaas/templates/blob/master/template/python3/Dockerfile) can be used to additional Alpine `apk` packages to be installed during build process. To install the [`ca-certificates`](https://pkgs.alpinelinux.org/package/edge/main/x86_64/ca-certificates) package for your `python3` function, you can specify

```yaml
build_options:
- ca-certificates
```

Important note: that the configuration of this value is dependent on the language template.  The template author must specify one or more [`ARG`](https://docs.docker.com/engine/reference/builder/#arg) in the `Dockerfile`.

#### Function: Environmental variables

You can set configuration via environmental variables either in-line within the YAML file or in a separate external file. Do not store confidential or private data in environmental variables. See: secrets.

* Define environment in-line within the file:

Imagine you needed to define a `http_proxy` variable to operate within a corporate network:

```yaml
functions:
  url-ping:
    lang: python
    handler: ./sample/url-ping
    image: alexellis2/faas-urlping
    environment:
      http_proxy: http://proxy1.corp.com:3128
      no_proxy: http://gateway/
```

* environment_file - defined in zero to many external files

```yaml
  environment_file:
    - file1.yml
    - file2.yml
```

If you specify a variable such as "rss_feed_url" in more than one `environment_file` file then the last file in the list will take priority.

Environment file format:

```yaml
environment:
  rss_feed_url: key1
  include_images: key2
```

> Note: external files take priority over in-line environmental variables. This allows you to specify a default and then have overrides within an external file.

#### Function: Secure secrets

OpenFaaS functions can make use of secure secrets using the secret store from Kubernetes or Docker Swarm. This is the recommended way to store secure access keys, tokens and other private data.

Create the secret with your orchestration tool i.e. `kubectl` or `docker secret create` then list the secret name as part of an array of secrets.

```yaml
secrets:
  - s3_access_key
  - s3_secret_key
```

#### Function: Read-Only Root Filesystem

The `readonly_root_filesystem` indicates that the function file system will be set to read-only except for the temporary folder `/tmp`.  This prevents the function from writing to or modifying the filesystem (e.g. system files). This is used to provide tighter security for your functions. You can set this value as a boolean:

```yaml
readonly_root_filesystem: true
```

This an optional boolean field, set to `false` by default.

#### Function: Constraints

Constraints are passed directly to the underlying container orchestrator. They allow you to pin a function to certain host or type of host.

Here is an example of picking only hosts with a Linux OS in Docker Swarm:

```yaml
   constraints:
     - "node.platform.os == linux"
```

Or only using nodes running with Windows:

```yaml
   constraints:
     - "node.platform.os == windows"
```

#### Function: Labels

Labels can be applied through a map which is passed directly to the container scheduler.
Labels are also available from the OpenFaaS REST API for querying or grouping functions.

Example of using a label to group by user or apply a `canary` label:

```yaml
   labels:
     canary: true
     Git-Owner: alexellis
```

> Important note: When used with a Kubernetes provider, labels support a restricted character set and length.
*"Valid label values must be 63 characters or less and must be empty or begin and end with an alphanumeric character
([a-z0-9A-Z]) with dashes (-), underscores (_), dots (.), and alphanumerics between."*
>
>See [Syntax and character set](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)
for more information

#### Function: Annotations

Annotations are a collection of meta-data which is stored with the function by the provider.
Annotations are also available from the OpenFaaS REST API for querying.

Example of setting a "topic" for the Kafka event connector:

```yaml
   annotations:
     topic: "kafka.payments-received"
     expire-date: "Wed Aug  8 07:40:18 BST 2018"
```

Example of setting a custom HTTP health check path and initial check delay:

```yaml
   annotations:
     com.openfaas.health.http.path: "/healthz"
     com.openfaas.health.http.initialDelay: "30s"
```

#### Function: Memory/CPU limits

Applying memory and CPU limits can be done through the `limits` and `requests` [fields](https://godoc.org/github.com/openfaas/faas-cli/stack#FunctionResources). It is advisable to always set a limit for your functions to prevent them consuming too many resources in your system.

> Important note: The value for memory for Kubernetes needs to be in the format "Mi" and for Docker Swarm it must be in the format "m"

Here we constrain the url-ping function to only use 40Mb of RAM at a maximum.

```YAML
  url-ping:
    lang: python
    handler: ./sample/url-ping
    image: alexellis/faas-url-ping:0.2
    limits:
      memory: 40Mi
    requests:
      memory: 20Mi
```

Here we constrain a function to use only `100m` which is equivalent to 1/10 of [an Intel Hyperthread core](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/).

```YAML
  url-ping:
    lang: python
    handler: ./sample/url-ping
    image: alexellis/faas-url-ping:0.2
    limits:
      cpu: 100m
    requests:
      cpu: 100m
```

The meanings and formats of `limits` and `requests` may vary depending on whether you are using Kubernetes or Docker Swarm. In general:

 - Requests ensures the stated host resource is available for the container to use
 - Limits specify the maximum amount of host resources that a container can consume

See docs for [Docker Swarm](https://docs.docker.com/config/containers/resource_constraints/) or for [Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#how-pods-with-r    esource-limits-are-run).

### YAML - environment variable substitution

The YAML stack format supports the use of `envsubst`-style templates. This means that you can have a single file with multiple configuration options such as for different user accounts, versions or environments.

Here is an example use-case, in your project there is an official and a development Docker Hub username/account. For the CI server images are always pushed to `exampleco`, but in development you may want to push to your own account such as `alexellis2`.

```yaml
functions:
  url-ping:
    lang: python
    handler: ./sample/url-ping
    image: ${DOCKER_USER:-exampleco}/faas-url-ping:0.2
```

Use the default (exampleco):

```sh
$ faas-cli build
$ DOCKER_USER="" faas-cli build
```

Override with "alexellis2" through an environment variable:

```sh
$ DOCKER_USER="alexellis2" faas-cli build
```

