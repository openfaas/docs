## Contribute to the project

There are many ways to contribute to the OpenFaaS project.

Before raising a PR or an issue request that you read our [Contributing guide](https://github.com/openfaas/faas) which applies to every OpenFaaS GitHub repository. We have a wide range of suggestions for contributing to the project and community and only some of those involve writing code.

### Five practical ideas for a first contribution

* Try the [workshop](https://github.com/openfaas/workshop)

* Read the [architecture diagrams](https://docs.openfaas.com/architecture/gateway/)

* Submit a function to the [Function Store](https://github.com/openfaas/store)

* Write a blog post and add it to our [community listing](https://docs.openfaas.com/community/#community-resources)

* Improve the [OpenFaaS CLI tooling](https://github.com/openfaas/faas-cli) with a PR or documentation

### Main Git repositories

Git is used for version control and all repositories are public available under two organisations.

#### OpenFaaS

OpenFaaS started as a single mono-repo called `faas` and has been broken out into separate repositories. For this reason you should always collate Issues, PRs, contributor counts, stars and similar statistics from the organisation as a whole. The data in the `faas` repo is not representive of the whole project.

| Repository       | Headline                        |
|------------------|---------------------------------|
| faas              | Main repository for project issues, suggestions, documentation and roadmap/backlog items. Also includes UI portal and API gateway | 
| certifier         | End-to-end tests written in Go for verifying OpenFaaS with Swarm or Kubernetes after a release, this also runs through CI for the `faas` repo      |
| faas-swarm        | Docker Swarm provider for OpenFaaS contains a stack file for deployment |
| faas-netes        | Kubernetes provider for OpenFaaS contains YAML and helm for deployment |
| faas-cloud        | An incubator project for a closed-loop CI/CD system between GitHub and OpenFaaS |
| faas-cli          | CLI for operating with OpenFaaS similar to `kubectl` or `docker` CLI    |
| templates         | Official templates for OpenFaaS CLI used to scaffold a new function |
| media             | Press-kit and media for the project branding and swag             |
| docs              | Official docs repository for this site - i.e. https://docs.openfaas.com             |
| workshop          | Hands-on training lab for learning OpenFaaS                                |
| nats-queue-worker | Asynchronous processing for deferred / queued work with OpenFaaS, based upon NATS Streaming |
| www               | Webpages for https://www.openfaas.com   |

https://github.com/openfaas/

#### OpenFaaS-Incubator

The incubator organisation is for experiments, research and for getting quick feedback. Only some of the projects incubated here will make it into the main project.

https://github.com/openfaas-incubator/

