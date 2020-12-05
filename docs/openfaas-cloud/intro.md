OpenFaaS Cloud is: the same OpenFaaS you know and love, but packaged as a multi-user platform with git integration, CI/CD, secrets and HTTPS included.

Read an introduction to [OpenFaaS Cloud on Alex Ellis' blog](https://blog.alexellis.io/introducing-openfaas-cloud/)

> OpenFaaS Cloud makes it even easier for developers to build and ship functions with Docker using a Git-based workflow with native integrations for GitHub and more integrations planned in the future.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Announcing <a href="https://twitter.com/openfaas?ref_src=twsrc%5Etfw">@openfaas</a> cloud at <a href="https://twitter.com/DevNetCreate?ref_src=twsrc%5Etfw">@DevNetCreate</a> <a href="https://twitter.com/hashtag/TeamServerless?src=hash&amp;ref_src=twsrc%5Etfw">#TeamServerless</a> <a href="https://twitter.com/hashtag/Serverless?src=hash&amp;ref_src=twsrc%5Etfw">#Serverless</a> <a href="https://t.co/n6hhcRK0I5">pic.twitter.com/n6hhcRK0I5</a></p>&mdash; Jock Reed (@JockDaRock) <a href="https://twitter.com/JockDaRock/status/983779290100613120?ref_src=twsrc%5Etfw">April 10, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

### Features

* Deploy and manage functions through `git push`
* Use the Docker Hub or your own private registry
* Use any official OpenFaaS language template or Docker images
* Add secrets to your functions to access services securely via Bitnami's SealedSecrets
* Secure integration with GitHub.com or self-hosted GitLab
* Personal dashboard for each user or organization with authz by OAuth2 and detailed metrics
* Auditing of events to Slack or custom function
* Runtime-logs for your functions
* Fast, non-root image builds using [buildkit](https://github.com/moby/buildkit/) from Docker

Check out the roadmap in the GitHub repo for what's coming next and how to get involved.

* See also: [openfaas/openfaas-cloud on GitHub](https://github.com/openfaas/openfaas-cloud)

### Who is OpenFaaS Cloud for?

OpenFaaS Cloud is for anyone who wants to focus on shipping functions without worrying about the CI/CD pipeline or underlying infrastructure. OpenFaaS comes in two flavours - a free community-run hosted version and self-hosted on your own cluster. 

|                | OpenFaaS                    | OpenFaaS Cloud                                  |
|:---------------|:----------------------------|:------------------------------------------------|
| Installation   |   Helm, Docker YAML         | GitHub App / GitLab tag |
| RBAC           |   Shared team / single user | Multi-user with OAuth2 |
| Administration |  faas-cli, API, UI          | "git-push" or GitHub UI |
| Policy         |  Specify in stack.yml       | Limits set for CPU/memory, read-only filesystem, non-root users |
| CI/CD          |  Jenkins, Travis, etc       | Built-in (via Buildkit)  |
| UI             |  OpenFaaS Portal            | Personal dashboard    |
| URLs           |  Gateway                    | Personal sub-domains    |
| TLS            |  Custom solution            | Built-in via LetsEncrypt and cert-manager |
| Source control |  Any                        | GitHub.com & GitLab self-hosted    |
| Secrets        |  Kubernetes/Swarm secrets   | Bitnami SealedSecrets    |

#### Self-hosted

OpenFaaS Cloud is open source software which you can use to host your OpenFaaS Cloud cluster. OpenFaaS Cloud brings a managed, multi-user experience with built-in dashboard and CI/CD. This lowers the barrier to entry for teams, meaning that developers only need to know how to use git and require no pre-assumed knowledge of Docker or Kubernetes.

* For an automated quick-start use [ofc-bootstrap](https://github.com/openfaas-incubator/ofc-bootstrap) to provision OpenFaaS Cloud in 100 seconds on Kubernetes [Video demo](https://www.youtube.com/watch?v=Sa1VBSfVpK0)
* Or start the [Developer guide](https://github.com/openfaas/openfaas-cloud/tree/master/docs) for a manual installation or to use Docker Swarm
