# YAML format reference

## Environmental variables/configuration

You can deploy non-encrypted secrets and configuration via environmental variables set either in-line or via external (environment) files.

> Note: external files take priority over in-line environmental variables. This allows you to specify a default and then have overrides within an external file.

Priority:

* environment_file - defined in zero to many external files

```yaml
  environment_file:
    - file1.yml
    - file2.yml
```

If you specify a variable such as "access_key" in more than one `environment_file` file then the last file in the list will take priority.

Environment file format:

```yaml
environment:
  access_key: key1
  secret_key: key2
```

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

## Constraints

Constraints work with Docker Swarm and are useful for pinning functions to certain hosts.

Here is an example of picking only Linux:

```yaml
   constraints:
     - "node.platform.os == linux"
```

Or only Windows:

```yaml
   constraints:
     - "node.platform.os == windows"
```

## Labels

Labels can be applied through a map which may be consumed by the back-end scheduler such as Docker Swarm or Kubernetes.

For example:

```yaml
   labels:
     kafka.topic: topic1
     canary: true
```

## Limits and Requests

Set limits and requests by setting the `limits` and `requests` objects in the yaml file, which is of type [FunctionResources](https://godoc.org/github.com/openfaas/faas-cli/stack#FunctionResources).

```
  url-ping:
    lang: python
    handler: ./sample/url-ping
    image: alexellis/faas-url-ping:0.2
    limits:
      memory: 40m
      cpu: 100m
    requests:
      memory: 40m
      cpu: 100m
```

The meanings of `limits` and `requests` vary depending on what orchestrator you are deploying on. In general:
 - Reserve maintains the host resources to ensure that the container can use them
 - Limits specify the maximum amount of host resources that a container can consume

#### Docker Swarm
 - Limit CPU is how many CPUs are allocated, e.g. `"1.5"`
 - Limit memory is the max amount of memory a container can use, e.g. `"4m"`
 - Reserve CPU -- unsure but may be `--cpu-shares`
 - Reserve Memory is a soft limit which is activated when Docker detects contention or low memory on the host machine

[Read more here](https://docs.docker.com/config/containers/resource_constraints/)

#### Kubernetes
 - Limit CPU is the total amount of CPU time that a container can use every 100ms
 - Limit memory is the same as Docker Swarm
 - Request CPU is the value sent to the [--cpu-shares](https://docs.docker.com/engine/reference/run/#cpu-share-constraint) flag of the `docker run` command
 - TODO Request Memory

[Read more here](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#how-pods-with-r    esource-limits-are-run)

## Other YAML fields

The possible entries for functions are documented below:

```yaml
functions:
  deployed_function_name:
    lang: node or python (optional)
    handler: ./path/to/handler (optional)
    image: docker-image-name
    environment:
      env1: value1
      env2: "value2"
    labels:
      label1: value1
      label2: "value2"
   constraints:
     - "com.hdd == ssd"
```

Use environmental variables for setting tokens and configuration.

