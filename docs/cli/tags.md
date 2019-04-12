# Working with image tags

All OpenFaaS functions are built into immutable Docker images before deployment. You can take advantage of Docker "tags" to organise your versions the CLI can also generate tags based upon Git metadata.

## Option 1 - YAML file

You can define your tag in the stack YAML file. By default this tag is ":latest"

```
image: my-fn
```

or 


```
image: my-fn:latest
```

## Option 2 - `--tag` option

The `--tag` option works with the `faas-cli` sub-commands: `build`, `push` and `deploy`.

When using a --tag feature which relies on metadata from a Git commit then the build, push and deploy commands must be run pointing at the same Git commit.

Example usage:

```
faas-cli build --tag=sha|branch
faas-cli push --tag=sha|branch
faas-cli deploy --tag=sha|branch
```

There are currently two formats for "automatic tags".

### 2.1 Use the SHA (`--tag=sha`)

In this example whatever tag is defined in your YAML file (or latest, if not is given) will be suffixed with "-" plus the short Git SHA. A Git repository will be required to use this feature.

Example:

```
image: my-fn:0.2

image: my-other-fn
```

Gives the equivalent:


```
image: my-fn:0.2-cf59cfc

image: my-other-fn:latest-cf59cfc
```

### 2.2 Use the SHA plus the branch (`--tag=branch`)

In this example you will get an output which includes the SHA and the branch name. This is useful for promotion code through enviroments with a continuous delivery tool. If you use one branch per environment in Git then the tool can parse the tag and match it to an environment.


Example:

(on the master branch)

```
image: my-fn:0.2
```

Gives the equivalent:

```
image: my-fn:0.2-master-cf59cfc
```

(on the staging branch)

```
image: my-fn:0.2-staging-cf59cfc
```

