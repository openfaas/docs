## OpenFaaS Cloud Architecture

The OpenFaaS Cloud Architecture in made up of several modules:

* A CI/CD pipeline made up of a number of OpenFaaS functions
* A dashboard (served through OpenFaaS)
* A container builder (buildkit)
* An authentication service (edge-auth)
* An edge-router used to map user sub-domains to endpoints (edge-router)

### Conceptual diagram

![](https://github.com/openfaas/openfaas-cloud/raw/master/docs/conceptual-overview.png)

This conceptual diagram shows how OpenFaaS Cloud integrates with GitHub.com/GitLab through the use of an event-driven architecture.

Main use-cases:

* User pushes code - GitHub/GitLab push event is sent to github-event/gitlab-event function triggering a CI/CD workflow
* User removes GitHub/GitLab app from one or more repos - garbage collection is invoked removing 1-many functions
* User accesses function via router using "pretty URL" format and request is routed to function via API Gateway

See [COMPONENTS.md](https://github.com/openfaas/openfaas-cloud/blob/master/docs/COMPONENTS.md) in the code repository for more information.

