## Code repositories

All source is organised under two repositories on GitHub: openfaas and openfaas-incubator.

### OpenFaaS org

OpenFaaS started as a single mono-repo called `faas` and has been broken out into separate repositories in the [openfaas](https://github.com/openfaas/) organisation.

For this reason you should always collate statistics from the `openfaas` organisation, rather than the repository of a single component, which would provide invalid data. So whether counting issues, PRs, contributors, stars, or any other metric, use the whole organisation.

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [docs](https://github.com/openfaas/docs)              | Official docs repository for this site - i.e. https://docs.openfaas.com             |
| [faas-cli](https://github.com/openfaas/faas-cli)          | CLI for operating with OpenFaaS similar to `kubectl` or `docker` CLI    |
| [faas](https://github.com/openfaas/faas)              | Main repository for project issues, suggestions, documentation and roadmap/backlog items. Also includes UI portal and API gateway |
| [faas-netes](https://github.com/openfaas/faas-netes)        | Kubernetes provider for OpenFaaS contains YAML and helm for deployment |
| [faasd](https://github.com/openfaas/faasd)              | Alternative to `faas-netes` - OpenFaaS for containerd, for a single node. |
| [templates](https://github.com/openfaas/templates)         | Official templates for OpenFaaS CLI used to scaffold a new function |
| [nats-queue-worker](https://github.com/openfaas/nats-queue-worker) | Asynchronous processing for deferred / queued work with OpenFaaS, based upon NATS Streaming |
| [certifier](https://github.com/openfaas/certifier)         | End-to-end tests written in Go for verifying OpenFaaS with Kubernetes or faasd after a release, this also runs through CI for the `faas` repo |
| [of-watchdog](https://github.com/openfaas/of-watchdog)              | The OpenFaaS watchdog re-written with mode-abstractions for both STDIO & HTTP |
| [classic-watchdog](https://github.com/openfaas/of-watchdog)              | The OpenFaaS watchdog re-written with mode-abstractions for both STDIO & HTTP |
| [openfaas.github.io](https://github.com/openfaas/openfaas.github.io)               | Source for https://www.openfaas.com and blog |
| [store](https://github.com/openfaas/store)             | Function and template store manifests |
| [media](https://github.com/openfaas/media)             | Press-kit and media for the project branding and merchandise             |
| [ingress-operator](https://github.com/openfaas/ingress-operator/) | Provides `FunctionIngress` CRD for Custom Domains on Kubernetes |
| [connector-sdk](https://github.com/openfaas/connector-sdk)         | Build your own event connectors for OpenFaaS |
| [workshop](https://github.com/openfaas/workshop)             | Practical training and hands-on labs for learning OpenFaaS |
| [vcenter-connector](https://github.com/openfaas-incubator/vcenter-connector) | Trigger OpenFaaS Functions from events in VMware vCenter |
| [faas-memory](https://github.com/openfaas-incubator/faas-memory) | An OpenFaaS Provider example using memory for state. |

### Deprecated

No longer maintained, source available.

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [faas-swarm](https://github.com/openfaas/faas-swarm)        | Docker Swarm provider for OpenFaaS contains a stack file for deployment |
| [openfaas-cloud](https://github.com/openfaas/openfaas-cloud)        | OpenFaaS Cloud - portable, multi-user Serverless Functions powered by GitOps |
| [ofc-bootstrap](https://github.com/openfaas/ofc-bootstrap) | "one-click" CLI to install OpenFaaS Cloud on Kubernetes |
| [faas-federation](https://github.com/openfaas-incubator/faas-federation) | Federate two OpenFaaS installations into one API |

### Training & tutorials

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [workshop-vscode](https://github.com/openfaas-incubator/workshop-vscode) | Run a Kubernetes workshop with VSCode in the browser |
| [openfaas-linkerd-workshop](https://github.com/openfaas/openfaas-linkerd-workshop) | Lightweight Serverless on Kubernetes with mTLS and traffic-splitting with Linkerd2 |
| [openfaas-function-auth](https://github.com/openfaas-incubator/openfaas-function-auth) | Examples of authentication in OpenFaaS Serverless functions. |

### Templates

| Repository        | Headline                         |
|:------------------|:---------------------------------|
| [templates](https://github.com/openfaas/templates)         | Official templates for OpenFaaS CLI used to scaffold a new function |
| [golang-http-template](https://github.com/openfaas/golang-http-template) | Golang template providing additional control over the HTTP request and response.|
| [powershell-http](https://github.com/openfaas-incubator/powershell-http-template) | PowerShell HTTP template |
| [ruby-http](https://github.com/openfaas-incubator/ruby-http) | A Ruby HTTP template for OpenFaaS |
| [python-flask-template](https://github.com/openfaas/python-flask-template) | OpenFaaS templates for Python 2.7/3.6 with Flask |
| [python3-debian](https://github.com/openfaas-incubator/python3-debian) | Template for Python3 on Debian for data-science / compiled pip modules |
| [perl-template](https://github.com/openfaas-incubator/perl-template) | Perl template for OpenFaaS |

