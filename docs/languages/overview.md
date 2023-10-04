## Languages overview

OpenFaaS supports various different languages through the use of its own templates concept.

Each and every function deployed to OpenFaaS is packaged in a Open Container Initiative (OCI) format container image. The job of a template is to help you create a container image, whilst abstracting away most of the boiler-plate code and implementation details.

A template contains a Dockerfile, along with an entrypoint, which are always hidden from the user. The user, provides code in a handler file and a package management manifest like a go.mod, requirements.txt or package.json file.

There are many community templates, of varying levels of support and maintenance. The official templates are maintained by OpenFaaS Ltd for commercial users, and anything else is maintained by the community.

### Official templates

There are a number of official templates maintained and recommended by OpenFaaS Ltd, the following are currently documented:

* [Go](/languages/go)
* [Node](/languages/go)
* [Python](/languages/go)
* [Dockerfile](/languages/dockerfile)

See also: [other templates information - C#, Java, Ruby](/reference/templates)

Community support:

* [PHP](/languages/php)

Additional templates are available from the community via `faas-cli template store list`

### Custom templates

You can also [create your own custom templates](/languages/custom), or fork an existing template and adapt it for your own needs.

### Existing Dockerfiles / images

You can bring along your own [pre-existing Dockerfiles and container images](/languages/dockerfile), so long as they conform to the [OpenFaaS workloads contract](/reference/workloads). You may need to add a health or readiness endpoint to make sure that no requests are lost during scaling up and down of your function.

