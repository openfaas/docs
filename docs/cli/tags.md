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
faas-cli build --tag=sha|branch|describe
faas-cli push --tag=sha|branch|describe
faas-cli deploy --tag=sha|branch|describe
```

There are currently three formats for "automatic tags".

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

In this example you will get an output which includes the SHA and the branch name. This is useful for promotion code through environments with a continuous delivery tool. If you use one branch per environment in Git then the tool can parse the tag and match it to an environment.


Example:

(on the master branch)

```
image: my-fn:0.2

image: my-fn:latest
```

Gives the equivalent:

```
image: my-fn:0.2-master-cf59cfc

image: my-fn:latest-master-cf59cfc
```

(on the staging branch)

```
image: my-fn:0.2-staging-cf59cfc

image: my-fn:latest-staging-cf59cfc
```

### 2.3 Use the output of `git describe` (`--tag=describe`)

In this example you will get the output of `git describe --tags --always`. This returns a human readable name based on an available git ref. Using `--tags`, means that the output which can be read as `<most-recent-parent-tag>-<number-of-commits-to-that-tag>-g<short-sha>`. Using `--always`, means that if the repo does not use tags, then we will still get the short SHA as output, matching `--tag=sha`

Example:

If you set the image name as

```
image: my-fn:0.2

image: my-fn:latest
```

Then, if you have no tags in the repo, `faas-cli` produces the equivalent:

```
image: my-fn:0.2-b0da13a

image: my-fn:latest-cfb0da13a59cfc
```

If you have just tagged the current commit in the repo, then `faas-cli` produces the equivalent:

```
image: my-fn:0.2-v0.1.0

image: my-fn:latest-v0.1.0
```
Note that the middle value, `<number-of-commits-to-that-tag>` is omitted when it is 0, that is, when you build that tag.

If there have been several commits (e.g. 3) since the most recent tag, then `faas-cli` produces the equivalent:

```
image: my-fn:0.2-v0.1.0-1-gb0da13a

image: my-fn:latest-v0.1.0-3-gb0da13a
```

A word of caution, this can only reference the tags that your local repository knows of, make sure to run `git fetch --tags` to ensure that you have a copy of all remote tags.
