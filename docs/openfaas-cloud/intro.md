OpenFaaS Cloud is: GitOps for your functions with native GitHub integrations

Read an introduction to [OpenFaaS Cloud on Alex Ellis' blog](https://blog.alexellis.io/introducing-openfaas-cloud/)

> OpenFaaS Cloud makes it even easier for developers to build and ship functions with Docker using a Git-based workflow with native integrations for GitHub and more integrations planned in the future.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Announcing <a href="https://twitter.com/openfaas?ref_src=twsrc%5Etfw">@openfaas</a> cloud at <a href="https://twitter.com/DevNetCreate?ref_src=twsrc%5Etfw">@DevNetCreate</a> <a href="https://twitter.com/hashtag/TeamServerless?src=hash&amp;ref_src=twsrc%5Etfw">#TeamServerless</a> <a href="https://twitter.com/hashtag/Serverless?src=hash&amp;ref_src=twsrc%5Etfw">#Serverless</a> <a href="https://t.co/n6hhcRK0I5">pic.twitter.com/n6hhcRK0I5</a></p>&mdash; Jock Reed (@JockDaRock) <a href="https://twitter.com/JockDaRock/status/983779290100613120?ref_src=twsrc%5Etfw">April 10, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

### Features

* "`git push` and get functions"
* Use the Docker Hub or your own private registry
* Use any official OpenFaaS language template
* Get secrets in your functions securely with SealedSecrets
* Secure integration with GitHub via GitHub App and OAuth permissions
* Auditing of events to Slack

Check out the roadmap in the GitHub repo for what's coming next and how to get involved.

### Who is OpenFaaS Cloud for?

OpenFaaS Cloud is for anyone who wants to focus on shipping functions without worrying about the CI/CD pipeline or underlying infrastructure. OpenFaaS comes in two flavours - a free community-run hosted version and self-hosted on your own cluster. 


|         | OpenFaaS                 |OpenFaaS Cloud |
|:--------|:-------------------------|:--------------|
| Installation |   Helm, Docker YAML | GitHub App    |
| RBAC |   Shared team / single user | Multi-user    |
| Administration | faas-cli, API, UI | "git-push" or GitHub UI |
| Policy |  Specify in stack.yml | Default limits set, read-only filesystem |
| CI/CD |  Jenkins, Travis, etc | Built-in (via Buildkit)  |
| UI |  OpenFaaS Portal | Personal dashboard    |
| URLs |  Gateway | Personal sub-domains    |
| Source control |  Any | GitHub & GitLab    |
| Secrets |  Kubernetes/Swarm secrets | SealedSecrets    |

#### Community cluster

The community-run cluster means you can do a `git push` and get a free, TLS-enabled endpoint without thinking about servers or Kubernetes. We do everything for you

#### Self-hosted

When you host OpenFaaS Cloud yourself you are providing a "`git push` - get functions" for yourself and your team. All the code is fully open source and licensed under MIT. Follow the "dev" guide in the GitHub repository to set yourself up on Kubernetes or Docker Swarm.

### Resources:

Read the code on GitHub:

https://github.com/openfaas/openfaas-cloud
