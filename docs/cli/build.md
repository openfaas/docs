# Build functions

The OpenFaaS CLI supports various options for building a function.  

For details and examples run 

```bash
faas-cli build --help
```

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

## 4.0 Personalize your image tags through ```--tag```

In one of our stack.yml files we have this block:

```yaml
...
functions:
  example_name:
    lang: dockerfile
    handler: ./example_name
    image: example_name
...
```

We are concerned with `image: example_name` field and if we use:

```bash
faas-cli build -f ./stack.yml
faas-cli push -f ./stack.yml
faas-cli deploy -f ./stack.yml
docker image ls
```

under TAG field we will see `latest` this is the default tag.

How to personalize that tag?

Well we just need to add `:example_tag` to the image name above so our field looks like that `image: example_name:example_tag`:

```yaml
...
functions:
  example_name:
    lang: dockerfile
    handler: ./example_name
    image: example_name:example_tag
...
```

```bash
faas-cli build -f ./stack.yml
faas-cli push -f ./stack.yml
faas-cli deploy -f ./stack.yml
docker image ls
```

Now under TAG instead of `latest` we have `example_tag`.

To further personalize your naming every time you commit to github, your commit get unique sha which you can append to your TAG with the following command `--tag=sha` like so:

```bash
faas-cli build -f ./stack.yml --tag=sha
faas-cli push -f ./stack.yml --tag=sha
faas-cli deploy -f ./stack.yml --tag=sha
docker image ls
```

Now under TAG will see `example_tag` changed with `example_tag-example_sha`

Now we want to extend this even further by adding the branch in which we commit so we swap `sha` with `branch`
and it becomes `--tag=branch`:

```bash
faas-cli build -f ./stack.yml --tag=branch
faas-cli push -f ./stack.yml --tag=branch
faas-cli deploy -f ./stack.yml --tag=branch
docker image ls
```

The TAG field will be named `example_tag-example_branch-example_sha`.

 For more information about passing build arguments to Docker, please visit the [Docker documentation](https://docs.docker.com/engine/reference/commandline/build/)