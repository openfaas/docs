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

## 1.0 Apply build options

The OpenFaaS CLI enables functions to be built with different options, e.g. `dev`, `debug`, etc.

By default all templates provide a minimal build as this optimizes function image sizes. Where appropriate, 3rd-party dependencies can be specified via `requirements.txt`. In scenarios where third-party dependencies also require native (e.g. C/C++) modules,
like `libssh` in Ruby and `numpy` or `pandas` in Python, then `--build-option` can be used.

* How to use

The OpenFaaS CLI provides a `--build-option` flag which enables named sets of native modules to be specified for inclusion in the function build.  

There are two ways to achieve this:

```bash
faas-cli build --lang python3 --build-option dev [--build-option debug]
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

There may be scenarios where a single native module need to be added to a build.  A single-package build option could be added as described above.  Alternatively a package could be specified through a `--build-arg`.

```bash
faas-cli build --lang python3 --build-arg ADDITIONAL_PACKAGE=jq
```

In the event a `build-option` is set the effect will be cumulative:

```bash
faas-cli build --lang python3 --build-option dev --build-arg ADDITIONAL_PACKAGE=jq
```

The entries in the template's Dockerfile described in 1.0 above need to be present for this mode of operation.

## 3.0 Pass custom build arguments

You can pass `ARG` values to Docker via the CLI.

```bash
faas-cli build --build-arg ARGNAME1=argvalue1 --build-arg ARGNAME2=argvalue2
``` 

Remeber to add any `ARG` values to the template's Dockerfile:

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
