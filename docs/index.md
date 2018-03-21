## OpenFaaS - Serverless Functions Made Simple

[![Go Report Card](https://goreportcard.com/badge/github.com/openfaas/faas)](https://goreportcard.com/report/github.com/openfaas/faas) [![Build
Status](https://travis-ci.org/openfaas/faas.svg?branch=master)](https://travis-ci.org/openfaas/faas) [![GoDoc](https://godoc.org/github.com/openfaas/faas?status.svg)](https://godoc.org/github.com/openfaas/faas) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![OpenFaaS](https://img.shields.io/badge/openfaas-serverless-blue.svg)](https://www.openfaas.com)

![OpenFaaS Logo](https://blog.alexellis.io/content/images/2017/08/faas_side.png)

OpenFaaS (Functions as a Service) is a framework for building serverless functions with Docker and Kubernetes which has first class support for metrics. Any process can be packaged as a function enabling you to consume a range of web events without repetitive boiler-plate coding.

[![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/fold_left.svg?style=social&label=Follow%20%40openfaas)](https://twitter.com/openfaas)

**Highlights**

* Ease of use through UI portal and *one-click* install
* Write functions in any language for Linux or Windows and package in Docker/OCI image format
* Portable - runs on existing hardware or public/private cloud - [Kubernetes](https://github.com/openfaas/faas-netes) and Docker Swarm native
* [CLI](http://github.com/openfaas/faas-cli) available with YAML format for templating and defining functions
* Auto-scales as demand increases

> Serverless Functions Made Simple.

![Stack](https://pbs.twimg.com/media/DFrkF4NXoAAJwN2.jpg)

## Governance

OpenFaaS is an independent project created by [Alex Ellis](https://www.alexellis.io) which is now being built and shaped by a growing community of contributors. Project website: [openfaas.com](https://www.openfaas.com).

## Get started with OpenFaaS

![Portal](https://pbs.twimg.com/media/C7bkpZbWwAAnKsx.jpg)

*Pictured: API gateway portal - designed for ease of use*

Read the [deployment guide](./deployment/)

## Presentations

### TechFieldDay presentation (Dockercon EU)

15 minute overview with demos on Kubernetes and with Alexa - [HD YouTube video](https://www.youtube.com/watch?v=C3agSKv2s_w&list=PLlIapFDp305AiwA17mUNtgi5-u23eHm5j&index=1)

### SkillsMatter video presentation

Great overview of OpenFaaS features, users and roadmap

* [HD Video](https://skillsmatter.com/skillscasts/10813-faas-and-furious-0-to-serverless-in-60-seconds-anywhere)

### OpenFaaS presents to CNCF Serverless workgroup

* [Video and blog post](https://blog.alexellis.io/openfaas-cncf-workgroup/)

### Closing Keynote at Dockercon 2017

Functions as a Service or FaaS was a winner in the Cool Hacks contest for Dockercon 2017.

* [Watch my FaaS keynote at Dockercon 2017](https://blog.docker.com/2017/04/dockercon-2017-mobys-cool-hack-sessions/)

If you'd like to find the functions I used in the demos head over to the [faas-dockercon](https://github.com/alexellis/faas-dockercon/) repository.

**Background story**

This is my original blog post on FaaS from January: [Functions as a Service blog post](http://blog.alexellis.io/functions-as-a-service/)

### Community

Have you written a blog about OpenFaaS? Send a Pull Request to the community page below.

* [Read blogs/articles and find events about OpenFaaS](https://github.com/openfaas/faas/blob/master/community.md)

If you'd like to join OpenFaaS community Slack channel to chat with contributors or get some help - then send a Tweet to [@alexellisuk](https://twitter.com/alexellisuk/) or email alex@openfaas.com.

### Roadmap and contributing

OpenFaaS is written in Golang and is MIT licensed - contributions are welcomed whether that means providing feedback, testing existing and new feature or hacking on the source.

To get started you can read the [roadmap](https://github.com/openfaas/faas/blob/master/ROADMAP.md) and [contribution guide](https://github.com/openfaas/faas/blob/master/CONTRIBUTING.md) or:

* [Browse FaaS issues on Github](https://github.com/openfaas/faas/issues).
* [Browse FaaS-CLI issues on Github](https://github.com/openfaas/faas-cli/issues).

Highlights:

* New: Kubernetes support via [FaaS-netes](https://github.com/openfaas/faas-netes) plugin
* New: FaaS CLI and easy install via `curl` and `brew`
* New: Windows function support
* New: Asynchronous/long-running OpenFaaS functions via [NATS Streaming](https://nats.io/documentation/streaming/nats-streaming-intro/) - [Follow this guide](https://github.com/openfaas/faas/blob/master/guide/asynchronous.md)

### Other

Example of a Grafana dashboards linked to OpenFaaS showing auto-scaling live in action: [here](https://grafana.com/dashboards/3526)

![](https://pbs.twimg.com/media/C9caE6CXUAAX_64.jpg:large)

An alternative community dashboard is [available here](https://grafana.com/dashboards/3434)
