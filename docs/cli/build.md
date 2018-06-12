# Build functions

The OpenFaaS CLI supports various options for building a function.  

For details and examples run 

```bash
faas-cli build --help
```

## 1.0 Pass custom build arguments

You can pass build-time arguments to Docker with

```bash
faas-cli build --build-arg ARGNAME1=argvalue1 --build-arg ARGNAME2=argvalue2
``` 
 and use them in the template's Dockerfile with

 ```dockerfile
 ARG ARGNAME1
 ARG ARGNAME2
 ```

 For more information about passing build arguments to Docker, please visit the [Docker documentation](https://docs.docker.com/engine/reference/commandline/build/)

## 2.0 Apply build options

The OpenFaaS CLI allows you to run a build with different options, f.e. `dev`, `debug`, etc.

By default all templates are restricted to a minor build, which doesn't allow you to use third-party dependencies that require native (f.e C/C++) modules,
like `libssh` in Ruby, `numpy` or `pandas` in Python, etc.

* How to use

The OpenFaaS CLI provides a solution by running a build in a dev mode, adding all required native modules.  

You can do this with

```bash
faas-cli build --lang python3 --build-option dev [--build-option debug]
```

or in YAML:

```yaml
    build_options:
    - debug
    - dev
```

If you are building multiple functions, we recommend using YAML configuration instead of CLI flag, as the flag is going to be applied to all functions listed in the YAML file.

> Currently only python and ruby templates are edited to support the feature.

* Edit templates to support dev build

One may want to support dev build for a custom template or edit the list of additional packages.

In order to modify a template to support dev build option, you should edit the `template.yml` with the following:

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

and edit `Dockerfile` with

```dockerfile
# Add the following line
ARG ADDITIONAL_PACKAGE 

# Edit `RUN apk --no-cache add curl \` to the following
RUN apk --no-cache add curl ${ADDITIONAL_PACKAGE} \  

```
