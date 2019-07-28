## Contribute to the project

There are many ways to contribute to the OpenFaaS project.

Before raising a PR or an issue request that you read our [Contributing guide](https://github.com/openfaas/faas) which applies to every OpenFaaS GitHub repository. We have a wide range of suggestions for contributing to the project and community and only some of those involve writing code.

### Five practical ideas for a first contribution

* Try the [workshop](https://github.com/openfaas/workshop)

* Read the [architecture diagrams](https://docs.openfaas.com/architecture/gateway/)

* Submit a function to the [Function Store](https://github.com/openfaas/store)

* Write a blog post and add it to our [community listing](https://docs.openfaas.com/community/#community-resources)

* Improve the [OpenFaaS CLI tooling](https://github.com/openfaas/faas-cli) with a PR or documentation

### Get up to speed

If you are new with Docker, Kubernetes or Go and would like to learn or just improve your skills, you may find the following links useful:

* **Docker**
    * [Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)
    * [Docker Documentation](https://docs.docker.com)
* [Kubernetes Documentation](https://kubernetes.io/docs/home/?path=browse)
* **Golang**
    * [The Go Programming Language](https://golang.org)
    * [Go Documentation](https://golang.org/doc/)
    * Alex Ellis' [golang basics](https://blog.alexellis.io/tag/golang-basics/) blog posts
    * [The Case For Go](https://gist.github.com/ungerik/3731476) - a list of sources collected by Erik Unger
    * Another interesting blog post - [Why should you learn Go?](https://medium.com/@kevalpatel2106/why-should-you-learn-go-f607681fad65) by Keval Patel
    * Official Golang [GitHUb](https://github.com/golang) account
    * [The Go Programming Language (Addison-Wesley Professional Computing)](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440)
    * [The Clean Coder: A Code of Conduct for Professional Programmers (Robert C. Martin)](https://www.amazon.co.uk/Clean-Coder-Conduct-Professional-Programmers/dp/0137081073/ref=sr_1_1?s=books&ie=UTF8&qid=1543083898&sr=1-1&keywords=the+clean+coder)
* [Travis CI User Documentation](https://docs.travis-ci.com)

### Main Git repositories

Git is used for version control and all repositories are public available under two organisations.

#### OpenFaaS

OpenFaaS started as a single mono-repo called `faas` and has been broken out into separate repositories. For this reason you should always collate Issues, PRs, contributor counts, stars and similar statistics from the organisation as a whole. The metrics within the `faas` repository alone are not representative of the whole project.

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [faas](https://github.com/openfaas/faas)              | Main repository for project issues, suggestions, documentation and roadmap/backlog items. Also includes UI portal and API gateway |
| [faas-netes](https://github.com/openfaas/faas-netes)        | Kubernetes provider for OpenFaaS contains YAML and helm for deployment |
| [faas-swarm](https://github.com/openfaas/faas-swarm)        | Docker Swarm provider for OpenFaaS contains a stack file for deployment |
| [certifier](https://github.com/openfaas/certifier)         | End-to-end tests written in Go for verifying OpenFaaS with Swarm or Kubernetes after a release, this also runs through CI for the `faas` repo |
| [faas-cli](https://github.com/openfaas/faas-cli)          | CLI for operating with OpenFaaS similar to `kubectl` or `docker` CLI    |
| [templates](https://github.com/openfaas/templates)         | Official templates for OpenFaaS CLI used to scaffold a new function |
| [openfaas-cloud](https://github.com/openfaas/openfaas-cloud)        | OpenFaaS Cloud - portable, multi-user Serverless Functions powered by GitOps |
| [docs](https://github.com/openfaas/docs)              | Official docs repository for this site - i.e. https://docs.openfaas.com             |
| [workshop](https://github.com/openfaas/workshop)          | Official workshop - Hands-on labs for learning OpenFaaS |
| [nats-queue-worker](https://github.com/openfaas/nats-queue-worker) | Asynchronous processing for deferred / queued work with OpenFaaS, based upon NATS Streaming |
| [www](https://github.com/openfaas/www)               | Webpages for https://www.openfaas.com   |
| [media](https://github.com/openfaas/media)             | Press-kit and media for the project branding and swag             |

https://github.com/openfaas/

#### OpenFaaS Incubator

The [openfaas-incubator organisation](https://github.com/openfaas-incubator/) contains projects which are under active development.

Some people wrongly assume that the "incubator" organisation means that these projects have a special status, or that they are unsuitable for production. Projects that are unsuitable for production usage will say so in their readme file.

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [of-watchdog](https://github.com/openfaas-incubator/of-watchdog)              | The OpenFaaS watchdog re-written with mode-abstractions for both STDIO & HTTP |
| [faas-idler](https://github.com/openfaas-incubator/faas-idler)         | Scale OpenFaaS functions to zero replicas after specified period of inactivity   |
| [openfaas-operator](https://github.com/openfaas-incubator/openfaas-operator)         | The OpenFaaS CRD Operator for Kubernetes   |
| [kafka-connector](https://github.com/openfaas-incubator/kafka-connector)         | The Kafka connector connects OpenFaaS functions to Kafka topics   |
| [ofc-bootstrap](https://github.com/openfaas-incubator/ofc-bootstrap) | "one-click" CLI to install OpenFaaS Cloud on Kubernetes |
| [ingress-operator](https://github.com/openfaas-incubator/ingress-operator/) | Provides `FunctionIngress` CRD for Custom Domains on Kubernetes |
| [connector-sdk](https://github.com/openfaas-incubator/connector-sdk)         | Build your own event connectors for OpenFaaS |
| [vcenter-connector](https://github.com/openfaas-incubator/vcenter-connector) | Trigger OpenFaaS Functions from events in VMware vCenter |
| [faas-federation](https://github.com/openfaas-incubator/faas-federation) | Federate two OpenFaaS installations into one API |
| [faas-memory](https://github.com/openfaas-incubator/faas-memory) | An OpenFaaS Provider example using memory for state. |

##### Training & tutorials

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [workshop-vscode](https://github.com/openfaas-incubator/workshop-vscode) | Run a Kubernetes workshop with VSCode in the browser |
| [openfaas-linkerd2](https://github.com/openfaas-incubator/openfaas-linkerd2) | Lightweight Serverless on Kubernetes with mTLS and traffic-splitting with Linkerd2 |
| [openfaas-function-auth](https://github.com/openfaas-incubator/openfaas-function-auth) | Examples of authentication in OpenFaaS Serverless functions. |

##### Templates

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [node10-express-template](https://github.com/openfaas-incubator/node10-express-template) | Node.js 10 Express template providing additional context and control over the HTTP response from your function |
| [node10-express-service](https://github.com/openfaas-incubator/) | Template with Node.js and Express.js exposed fully |
| [node8-express-template](https://github.com/openfaas-incubator/node8-express-template) | Node.js 8 template for OpenFaaS with HTTP via Express.js |
| [golang-http-template](https://github.com/openfaas-incubator/golang-http-template) | Golang template providing additional control over the HTTP request and response.|
| [powershell-http](https://github.com/openfaas-incubator/powershell-http-template) | PowerShell HTTP template |
| [ruby-http](https://github.com/openfaas-incubator/ruby-http) | A Ruby HTTP template for OpenFaaS |
| [python-flask-template](https://github.com/openfaas-incubator/python-flask-template) | OpenFaaS templates for Python 2.7/3.6 with Flask |
| [python3-debian](https://github.com/openfaas-incubator/python3-debian) | Template for Python3 on Debian for data-science / compiled pip modules |
| [perl-template](https://github.com/openfaas-incubator/perl-template) | Perl template for OpenFaaS |
