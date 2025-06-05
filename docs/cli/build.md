# Build functions

The OpenFaaS CLI supports various options for building a function.  

For details and examples run 

```bash
faas-cli build --help
```

* Build images with Docker

The `faas-cli build` command builds a Docker image into your local Docker library, which can then be used locally or pushed into a remote Docker registry. Each change of your function requires a new `faas-cli build` command to be issued.

* How to do CI/CD

When it comes to continuous integration and delivery you can use the `faas-cli` tool on your build server to build and deploy your code using the built-in commands. 

* Generate a Dockerfile with `--shrinkwrap`

If you are using an alternative container image builder or are automating the `faas-cli` then you can use the `--shrinkwrap` flag which will produce a folder named `./build/function-name` with a Dockerfile. This bundle can be used with any container builder.

## Plugins and build-time secrets

!!! info "Experimental feature"

    This is an experimental feature which means that it may change in the future.

When using Docker's buildkit project to build your containers, faas-cli can pass in the arguments to mount different secrets into the build process.

Any other mechanism should be considered insecure because it will leak into the final image or the local image in one way or another.

For Go users, make use of vendoring. It's what we use and it means you do not have to resort to insecure practices like sharing Personal Access Tokens (PAT) between users.

Below we have an example for Python using the pip package manager and for node modules with npm. The approach is similar for different package managers.

1. Download and enable the OpenFaaS Pro plugin
2. Create a local file in the format required
3. Update a `build_secret` in `stack.yml` so it gets mounted into the container
4. Run `faas-cli pro build` or `faas-cli pro publish`, `faas-cli pro up` is not available at this time

### Private access to a Python pip repository

First enable OpenFaaS Pro:

```bash
faas-cli plugin get pro
faas-cli pro enable
```

Download the OpenFaaS Pro template using your customer credentials:

```bash
faas-cli template pull https://github.com/openfaas/customers

faas-cli new --list
Languages available as templates:
- python@3.8-debian
```

Create a new function from the template:

```bash
export OPENFAAS_PREFIX=docker.io/alexellis2

faas-cli new --lang python@3.8-debian \
  withprivate
```

Next, set up a build secret, for instance to fetch Pip modules from a private PyPi repository:

```yaml
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  withprivate:
    lang: python3-http
    handler: ./withprivate
    image: openfaasltd/withprivate:0.0.1
    build_secrets:
      pipconf: ${HOME}/.config/pip/pip.conf
```

Set up the private authentication for `pip.conf`:

```ini
[global]
index-url = https://aws:CODEARTIFACT_TOKEN@OWNER-DOMAIN.d.codeartifact.us-east-1.amazonaws.com/pypi/REPOSITORY/simple/
```

Then run a build with:

```bash
faas-cli pro build
```

The `faas-cli pro publish` command can also be used instead of `faas-cli pro build`.

Within a GitHub Action, the short-lived token associated to the job is used to verify your license for this feature.

Add to your workflow.yaml:

```yaml
    permissions:
      contents: 'read'
      id-token: 'write'
```

Then:

```bash
faas-cli plugin get pro
faas-cli pro enable

faas-cli pro build / publish
```

If you're cloning from a private Git repository, without using a private PyPi repository, then you can use the `.netrc` approach instead:

.netrc:

```
machine github.com
login username
password PAT
```

Then update stack.yml:

```yaml
functions:
  withprivate:
..
    build_secrets:
      netrc: ${HOME}/.netrc
```

Bear in mind that at this time, `GITHUB_TOKEN` in a GitHub Action cannot be used to clone other repositories, even within the same organisation.

### Private npm modules

Get the OpenFaaS Pro plugin and enable it:

```bash
faas-cli plugin get pro
faas-cli pro enable
```

Create a function:

```bash
export OPENFAAS_PREFIX=openfaasltd
faas-cli template store pull node22
faas-cli new --lang node22 withprivatenpm
```

You will need to create an authentication token to install private npm modules. These instructions will differ depending on the registry you want to use:

- For the npmjs registry you can generate a token using the npm cli, see: [Create a new access token](https://docs.npmjs.com/using-private-packages-in-a-ci-cd-workflow#create-a-new-access-token).
- For Github Packages, see: [Authenticating to GitHub Packages](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry#authenticating-to-github-packages)

Once you have an authentication token you can add a registry user account with the `npm login` command:

```bash
npm login --scope=@OWNER --registry=https://npm.pkg.github.com

> Username: USERNAME
> Password: TOKEN
> Email: PUBLIC-EMAIL-ADDRESS
```

Or edit your `~/.npmrc` file to include the authentication token for your npm registry.

```
//registry.npmjs.org/:_authToken=TOKEN
```

Add the `.npmrc` file as a build secret to the `stack.yml` file:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  withprivatenpm:
    lang: node22
    handler: ./withprivatenpm
    image: openfaasltd/withprivatenpm:latest
    build_secrets:
      npmrc: ${HOME}/.npmrc
```

Run a build with:

```bash
faas-cli pro build -f stack.yml
```

You'll also need an updated version of the node template to mount the secret passed in from the OpenFaaS Pro plugin. Update `template/node22/Dockerfile` and replace the second `npm i` command with:

```Dockerfile
RUN --mount=type=secret,id=npmrc,mode=0666,dst=/home/app/.npmrc npm i
```

## 1.0 Apply build options

The OpenFaaS CLI enables functions to be built with different options, e.g. `dev`, `debug`, etc.

By default all templates provide a minimal build as this optimizes function image sizes. Where appropriate, 3rd-party dependencies can be specified via `requirements.txt`. In scenarios where third-party dependencies also require native (e.g. C/C++) modules,
like `libssh` in Ruby and `numpy` or `pandas` in Python, then `--build-option` can be used.

* How to use

The OpenFaaS CLI provides a `--build-option` flag which enables named sets of native modules to be specified for inclusion in the function build.  

There are two ways to achieve this:

```bash
faas-cli build --lang python3-http --build-option dev [--build-option debug]
```

or in YAML:

```yaml
    build_options:
    - debug
    - dev
```

Where multiple functions are being built, the YAML configuration is recommended over use of the CLI flag, as the CLI flag applies the `--build-option` to all functions involved in the build activity.

> Currently, of the official templates, Python and Ruby templates include named build options.

* Edit templates to support additional build options

It is possible to amend build options in both official and custom templates.  

> Altering of official templates should be carefully considered in the context of repeatable builds

In order to modify a template to support further build options, edit the `template.yml` using the following pattern:

```yaml
build_options: 
  - name: dev
    packages: # A list of required packages
      - make
      - automake
      - gcc
      #- etc.
  - name: debug
    packages: 
      - mg
      - iw
      #- etc.
```

and if not already present edit `Dockerfile` with:

```dockerfile
# Add the following line
ARG ADDITIONAL_PACKAGE 

# Edit `RUN apk --no-cache add curl \` to the following
RUN apk --no-cache add curl ${ADDITIONAL_PACKAGE} \  

```
## 2.0 Pass ADDITIONAL_PACKAGE through `--build-arg`

There may be scenarios where a single native module needs to be added to a build.  A single-package build option could be added as described above.  Alternatively a package could be specified through a `--build-arg`.

```bash
faas-cli build --lang python3-http --build-arg ADDITIONAL_PACKAGE=jq
```

In the event a `build-option` is set the effect will be cumulative:

```bash
faas-cli build --lang python3-http --build-option dev --build-arg ADDITIONAL_PACKAGE=jq
```

The entries in the template's Dockerfile described in 1.0 above need to be present for this mode of operation.

## 3.0 Pass custom build arguments

You can pass `ARG` values to Docker via the CLI.

```bash
faas-cli build --build-arg ARGNAME1=argvalue1 --build-arg ARGNAME2=argvalue2
``` 

Remember to add any `ARG` values to the template's Dockerfile:

 ```dockerfile
 ARG ARGNAME1
 ARG ARGNAME2
 ```

 For more information about passing build arguments to Docker, please visit the [Docker documentation](https://docs.docker.com/engine/reference/commandline/build/)

## 4.0 Building with large function sets

Performing a `build` action against a `stack.yml` which contains a large suite of serverless function definitions will result in each of the defined functions being built.  The CLI makes available facilities that assist in this scenario.

The `--parallel` flag aims to reduce total build time by enabling more than one function build action to take place concurrently.  Additionally, there may be situations where building *all* the defined functions is undesirable - for example where only one of the functions has had its code updated.  In this instance the `--filter` and `--regex` flags can be used.

Consider a project with `fn1`, `fn2`, `fn3`, `fn22`, `fn33` functions all defined within a single YAML file.

### 4.1 Using the `--parallel` flag

Parallel enables the user to specify how many concurrent function build actions should be performed.  The default is that functions will be built serially, one after the other.

The following will see all the project functions' build actions performed concurrently:

```bash
faas-cli build --parallel 5
``` 

!!! note
    Remember to add -f if using a non-default yaml file: `faas-cli build --parallel 5 -f projectfile.yml`

Parallel can be combined with either of the `--filter` and `--regex` flags to parallel build a subset of the functions.

### 4.2 Using the `--filter` flag

Filter performs wildcard matching against function names in YAML file so that the build action will only be performed against those that match.

The following filter would build only `fn2` from `stack.yml`:

```bash
faas-cli build --filter "fn2"
``` 
Wildcards can be added using `*`.  The following will result in both `fn2` and `fn22` being built:

```bash
faas-cli build --filter "fn2*"
``` 

### 4.3 Using the `--regex` flag

Regex performs a similar action to `--filter` but allows for more complex patterns to be defined through regular expressions.

The following regex would result in `fn1`, `fn2` & `fn3` being built from the earlier project's `stack.yml`:

```bash
faas-cli build --regex "fn[0-9]$"
```

## Building multi-arch images for ARM and Raspberry Pi

If you're Raspberry Pi or ARM servers to run your OpenFaaS on Kubernetes or with faasd server, then you will need to use the `publish` command instead which uses emulation and in some templates cross-compilation to build an ARM image from your PC.

> It is important that you do not install Docker or any build tools on your faasd instance. faasd is a server to serve your functions, and should be treated as such.

The technique used for cross-compilation relies on Docker's [buildx](https://docs.docker.com/buildx/working-with-buildx/) extension and [buildkit project](https://docs.docker.com/develop/develop-images/build_enhancements/). This is usually pre-configured with Docker Desktop, and Docker CE when installed on an Ubuntu system.

First install the QEMU utilities which allow for cross-compilation:

```bash
$ docker run --rm --privileged \
  multiarch/qemu-user-static \
  --reset -p yes
```

Or if you're an arkade user, run `arkade install qemu-static`.

In addition, for CI, you can also add the `--reset-qemu` flag to `faas-cli publish`.

The faas-cli attempts to enable Docker's experimental flag for the CLI, but you may need to run the following, if you get an error:

```
export DOCKER_CLI_EXPERIMENTAL=enabled
```

Now run this command on your laptop or workstation, not on the Raspberry Pi:

```bash
faas-cli publish -f stack.yml --platforms linux/arm/v7
```

If you're running a 64-bit ARM OS like Ubuntu on an AWS Graviton or Raspberry Pi 4, then use:

```bash
faas-cli publish -f stack.yml --platforms linux/arm64
```

You can also add multiple platforms to publish an image which will run on an ARM device, and on a regular Intel host:

```bash
faas-cli publish -f stack.yml --platforms linux/arm/v7,linux/amd64
```

Then deploy the function:

```bash
faas-cli deploy -f stack.yml
```