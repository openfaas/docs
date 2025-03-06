# OpenFaaS YAML file reference

OpenFaaS has its own YAML file called a "stack file" which is used to provide configuration for functions.

This page is the reference guide to the schema and how to use each field.

Configuration is split between:

* build-time - how to build a container from the source provided
* deploy time - how to deploy the function to and in OpenFaaS

!!! tip "Generate Kubernetes resources"
    Did you know that the OpenFaaS YAML files can also be converted into Kubernetes resources using `faas-cli generate`?

    If you use a GitOps tool like Argo or Flux, you can retain your `stack.yml` file for building functions, and testing locally, then generate a CustomResource when required with: `faas-cli generate | kubectl apply -f`

## Functions belong together

The YAML file can hold one to many functions separated by separate entries.

Example:

```bash
$ faas-cli new --lang go fn1
$ faas-cli new --lang go fn2 --append=fn1.yml
```

Produces:

```yaml
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

## Provider

The only valid value for provider `name` is `openfaas`.

### Gateway

The gateway URL can be hard-coded into the YAML file or overridden at deployment time with the --gateway flag or OPENFAAS_URL env-var.

## Functions

The `functions` element holds a map of functions, by default all functions are acted on with CLI verbs, but you can filter them with `--filter` or `--regex`.

### Function Name

The function Name is specified by a key in the functions map, i.e. `fn1` in the above example. Function name must be unique within a `stack.yml` file.

Valid function names follow ietf [rfc1035](https://tools.ietf.org/html/rfc1035) which is also used for DNS sub-domains.

> (DNS_LABEL): An alphanumeric (a-z, and 0-9) string, with a maximum length of 63 characters, with the '-' character allowed anywhere except the first or last character, suitable for use as a hostname or segment in a domain name.

### Function: Language

The `lang` field refers to which template is going to be used to build the function. The templates are expected to be found in the ./template folder and will be pulled from GitHub if not present.

### Function: Handler

The function `handler` field refers to a folder where the function's source code can be found, it must always be a folder and not a filename.

### Function: Image

The `image` field refers to a Docker image reference, this could be on the Docker Hub, in your local Docker library or on another remote server.

### Function: Skip build

The `skip_build` field controls whether the CLI will attempt to build the Docker image for the function.  When `true`, the build step is skipped and you should see a message printed to the terminal `Skipping build of: "function name"`.

This an optional boolean field, set to `false` by default.

### Function: Build Options

The `build_options` field can be used to pass a list of additional configurations for a template.

These must be pre-defined within the template and can be used to populate the `ADDITIONAL_PACKAGE` field in the Dockerfile used by the template.

For instance, here's an example from the `python3` template which is based upon Alpine Linux.

> Note: if you want to install Python development packages, you may find that the `python3-debian` template is a better fit, since it comes with build tools pre-installed.

```yaml
language: python3
fprocess: python3 index.py
build_options:
  - name: mysql
    packages: 
      - mysql-client
      - mysql-dev
  - name: pillow
    packages: 
      - jpeg-dev
      - zlib-dev
      - freetype-dev
      - lcms2-dev
      - openjpeg-dev
      - tiff-dev
      - tk-dev
      - tcl-dev
      - harfbuzz-dev
      - fribidi-dev
```

Given the template defines a `mysql` and `pillow` build option, you can add either or both of them to your stack.yml file so that these preconfigured packages are installed at build time.

```yaml
build_options:
- ca-certificates
```

The packages listed will be expounded into the [Dockerfile](https://github.com/openfaas/templates/blob/master/template/python3/Dockerfile) at build-time via the `ADDITIONAL_PACKAGE` environment variable.

```Dockerfile
FROM --platform=${TARGETPLATFORM:-linux/amd64} python:3-alpine

# Allows you to add additional packages via build-arg
ARG ADDITIONAL_PACKAGE

# Install packages
RUN apk --no-cache add ca-certificates ${ADDITIONAL_PACKAGE}
```

If you don't want to or cannot update the template, then you can pass the `ADDITIONAL_PACKAGE` directly instead, see the next section.

### Function: Build Args (`build-args`)

A map of build args can be passed to the container builder. These are compatible with Docker, BuildKit and Kaniko. Other containers builders may vary in their support.

An example of a build argument may be for enabling Go modules, or a HTTP_PROXY as per below:

```yaml
functions:
  with_go_modules:
    handler: ./with_go_modules
    lang: go
    build_args:
      HTTP_PROXY: http://squid.corp.ad.example.com
      GO111MODULE: on
```

These can also be passed via the CLI using `faas-cli build --build-arg key=value` or `faas-cli up --build-arg key=value`

### Function: Environmental variables

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

### Function: Secure secrets

OpenFaaS functions can make use of secure secrets using the secret store from Kubernetes or faasd. This is the recommended way to store secure access keys, tokens and other private data.

Create the secret with your orchestration tool i.e. `kubectl` or `docker secret create` then list the secret name as part of an array of secrets.

```yaml
secrets:
  - s3_access_key
  - s3_secret_key
```

### Function: Read-Only Root Filesystem

The `readonly_root_filesystem` indicates that the function file system will be set to read-only except for the temporary folder `/tmp`.  This prevents the function from writing to or modifying the filesystem (e.g. system files). This is used to provide tighter security for your functions. You can set this value as a boolean:

```yaml
readonly_root_filesystem: true
```

This an optional boolean field, set to `false` by default.

### Function: Constraints

Constraints are passed directly to the underlying container orchestrator. They allow you to pin a function to certain host or type of host.

Here is an example of picking only hosts with an example constraint for FIPS compliance:

```yaml
   constraints:
     - "example.com.node-restriction.kubernetes.io/fips=true"
```

The format is that of a Kubernetes  [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector).

First, update stack.yml:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  low:
    lang: go
    handler: ./low
    image: alexellis2/low:latest
    constraints:
    - "lowmem=true"
```

And run `faas-cli up` or `faas-cli deploy`.

You'll see an error like the following. So we need to add the `lowmem=true` label to one or more nodes.

```bash
kubectl get event -n  openfaas-fn -w
LAST SEEN   TYPE      REASON              KIND         MESSAGE
15m         Warning   FailedScheduling    Pod          0/3 nodes are available: 3 node(s) didn't match node selector.
```

Apply a label to the nodes or nodepool:

```bash
kubectl label node/primary-ofc-3irgb --overwrite lowmem=true
```

Check the label was applied:

```bash
kubectl get nodes -l lowmem=true
NAME                STATUS   ROLES    AGE    VERSION
primary-ofc-3irgb   Ready    <none>   136d   v1.19.3
```

Now deploy the code again, or wait for the scheduler to move the pod to the matching node.

```bash
kubectl get pod -n openfaas-fn -o wide
NAME                 READY   STATUS    IP             NODE
low-5fbb9fd6-t69q4   1/1     Running   10.244.5.121   primary-ofc-3irgb
```

### Function: Labels

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

### Function: Annotations

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

### Function: Memory/CPU limits

Applying memory and CPU limits can be done through the `limits` and `requests` [fields](https://godoc.org/github.com/openfaas/faas-cli/stack#FunctionResources). It is advisable to always set a limit for your functions to prevent them consuming too many resources in your system.

> Important note: The value for memory for Kubernetes needs to be in the format "Mi".

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

Read more for: [Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#how-pods-with-resource-limits-are-run).

## Configuration

The configuration section allows you to define additional configuration that is global to the entire stack, currently this mostly impacts function build time options.

### Templates

The `templates` list allows you to define the information required to pull the templates for your functions.  This list of templates will automatically be pulled when you build your functions. When configured correctly, this allows you to completely build your functions with just `faas-cli build`.  Without this section, you must manually `faas-cli template pull <source>` _before_ you use `faas-cli build`.

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  create-batch:
    lang: python3-http-debian
    handler: ./create-batch
    image: welteki2/create-batch:0.0.1

configuration:
  templates:
    - name: python3-http
      source: https://github.com/openfaas/python-flask-template
```

### Copy

The `copy` list allows you to define additional project paths that will be copied into your function's handler folder.

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  hello:
    lang: python3
    handler: ./hello
    image: openfaas/hello:0.1.0

configuration:
  copy:
    - ./common
    - ./data
    - ./models
```

Given the above configuration, the build folder for a python `hello` function would look like

```sh
build
└── hello
    ├── Dockerfile
    ├── function
    │   ├── __init__.py
    │   ├── models
    │   │   └── ...
    │   ├── data
    │   │   └── ...
    │   ├── common
    │   │   └── ...
    │   ├── handler.py
    │   └── requirements.txt
    ├── index.py
    ├── requirements.txt
    └── template.yml
```

The CLI `build` command also has a related flag `--copy-extra`.  When this flag is used, the paths specified by the flag will be _merged_ into the list from the YAML.  This means it will extend, not replace, the values specified in the file.

**Note:** These paths _must_ be subpaths of the project and _not equal_ to the entire project. For example, you can not reference `../` or `$HOME/.ssh`, any path outside of the current directory will be skipped.


## YAML - environment variable substitution

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

### YAML - template stack configuration

The `configuration` field stand alone and not part of the `function` field, add it to the top level of the YAML file.

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:

...

configuration:
  templates:
    - name: perl-alpine
    - name: rust
      source: https://github.com/openfaas-incubator/openfaas-rust-template
```

Pull templates listed under the `configuration.templates` field:

```sh
$ faas-cli template pull stack
```

By default if only `name` is provided the template will be pulled from the template store.

The templates will be automatically pulled during build time.

### YAML - pinning versions of templates

Templates may change over time, including breaking changes.

If you want additional stability, or have run into an issue with a newer version of a template, you can pin it.

You can pin a template to a specific release tag or branch by adding `#` plus the name required to the URL for the `source` field.

For example:

```yaml
configuration:
  templates:
    - name: golang-middleware
      source: https://github.com/openfaas/golang-http-template#0.7.0
```

Then run:

```sh
$ faas-cli template pull stack

Pulling template: golang-middleware from configuration file: stack.yml
Fetch templates from repository: https://github.com/openfaas/golang-http-template at 0.7.0
2022/09/21 16:18:56 Attempting to expand templates from https://github.com/openfaas/golang-http-template
2022/09/21 16:18:58 Fetched 2 template(s) : [golang-http golang-middleware] from https://github.com/openfaas/golang-http-template
```

As an alternative, you can fork any template and customise it or change it to suit your needs, and use that URL instead, even if it has the same name as the original template.

For example:

```yaml
configuration:
  templates:
    - name: golang-middleware
      source: https://github.com/alexellis/golang-http-template
```
