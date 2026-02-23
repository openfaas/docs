## OpenFaaS - Serverless Functions Made Simple

![OpenFaaS Logo](https://blog.alexellis.io/content/images/2017/08/faas_side.png)

OpenFaaS&reg; makes it easy for developers to deploy event-driven functions and microservices to Kubernetes without repetitive, boiler-plate coding. Package your code or an existing binary in a Docker image to get a highly scalable endpoint with auto-scaling and metrics.

[![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/fold_left.svg?style=social&label=Follow%20%40openfaas)](https://twitter.com/openfaas)

## Highlights

* Portable functions platform - run functions on any cloud or on-premises without fear of lock-in
* Write functions in any language and package them in Docker/OCI-format containers
* Easy to use - built-in UI, powerful CLI and one-click installation
* Scale as you go - handle spikes in traffic, and scale down when idle
* Ecosystem - community marketplace for functions and language templates

### Pick your version

* [OpenFaaS Standard/For Enterprises](/openfaas-pro/introduction/) for commercial use & production, deployed to Kubernetes
* Community Edition - free for personal use only, 60-day limit for commercial use, deployed to Kubernetes
* faasd - Free to use on any cloud or on-premises, works on a single VM (no clustering or Kubernetes)

![OpenFaaS Stack](https://github.com/openfaas/faas/raw/master/docs/of-layer-overview.png)
> Conceptual layers of the OpenFaaS stack

See also: [Tech stack & layers](/architecture/stack/) & [Preparing for production](/architecture/production/)

## Get started

Start out with one of the options from our self-service training range:

* [See our Official Training page](/tutorials/training/)

Or go ahead and deploy OpenFaaS straight to Kubernetes/OpenShift or to a VM using faasd:

![OpenFaaS Dashboard - overview page](https://docs.openfaas.com/images/dashboard/fn-overview.png)

![OpenFaaS Dashboard - invocation graphs](https://docs.openfaas.com/images/dashboard/invocation-graphs.png)

*Pictured: OpenFaaS Dashboard for [OpenFaaS Standard/For Enterprises](/openfaas-pro/dashboard)*

* [Deployment guides](./deployment/)

## Video presentations/demos

* [Build and scale a Python function with OpenFaaS (2023)](https://www.youtube.com/watch?v=igv9LRPzZbE)
* [Meet faasd. Look Ma’ No Kubernetes! 2020](https://www.youtube.com/watch?v=ZnZJXI377ak&feature=youtu.be)
* [Getting Beyond FaaS: The PLONK Stack for Kubernetes Developers 2019](https://www.youtube.com/watch?v=NckMekZXRt8&feature=emb_title)
* [Serverless Beyond the Hype - Alex Ellis - GOTO 2018](https://www.youtube.com/watch?v=yOpYYYRuDQ0)
* [How LivePerson is Tailoring its Conversational Platform Using OpenFaaS - Simon Pelczer 2019](https://www.youtube.com/watch?v=bt06Z28uzPA)
* [Digital Transformation of Vision Banco Paraguay with Serverless Functions @ KubeCon 2018](https://kccna18.sched.com/event/GraO/digital-transformation-of-vision-banco-paraguay-with-serverless-functions-alex-ellis-vmware-patricio-diaz-vision-banco-saeca)
* [Introducing "faas" - Cool Hacks Keynote at Dockercon 2017](https://blog.docker.com/2017/04/dockercon-2017-mobys-cool-hack-sessions/)

## Community

OpenFaaS has a large community of users, and a growing ecosystem of partners and integrations.

### Weekly Office Hours

We run a [weekly Office Hours call](/community/) for any user to join to ask questions, get help, and to share feedback.

### OpenFaaS for commercial and internal use

!!! info "Do we need the Community Edition (CE) or Pro?"
    The OpenFaaS (CE) is licensed for exploration or an initial PoC. OpenFaaS Standard/for Enterprises was specifically built for use in production.

    You can find out more about [OpenFaaS options here](/openfaas-pro/introduction) or [contact us to book a meeting](https://forms.gle/g6oKLTG29mDTSk5k9).

### OpenFaaS Adopters

If you're using OpenFaaS within your team, send a pull request to [ADOPTERS.md](https://github.com/openfaas/faas/blob/master/ADOPTERS.md) to let us know, or email us at: [contact@openfaas.com](mailto:contact@openfaas.com). You can also view customer logos on the [homepage](https://openfaas.com/).

## Governance

OpenFaaS was originally created in 2019 by [Alex Ellis](https://www.alexellis.io) as an open source project. It transitioned to a commercial model in 2019 and is worked on full time by employees of OpenFaaS Ltd. A free version is available for personal, non-commercial and hobbyist use in OpenFaaS CE and faasd CE.

OpenFaaS is hosted by OpenFaaS Ltd (registration: 11076587), a company which also offers commercial services, homepage sponsorships, and support. OpenFaaS &reg; is a registered trademark in England and Wales.

### Sponsored credits

Thank you to the following vendors for providing sponsored cloud credit for testing and enablement on their platform. Listed in order of contribution:

* [DigitalOcean](https://m.do.co/c/8d4e75e9886f)

If you'd like to sponsor the OpenFaaS homepage, or contribute cloud resources, [you can do so via GitHub](https://github.com/sponsors/openfaas) or by [contacting the team via email](mailto:contact@openfaas.com).
