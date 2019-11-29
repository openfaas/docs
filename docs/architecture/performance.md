## Notes on performance testing (or load-testing)

This page is designed to help you get a realistic and representative view of the performance of OpenFaaS whether you want to run a performance test, load-test or benchmark.

You may have started using OpenFaaS already or perhaps you have been asked to do some due diligence before starting a new project. Before proceeding please run through the project checklist to make sure your environment is properly tuned and that you are using *an appropriate function template tuned for performance*.

The default configuration for OpenFaaS targets a development-environment and not production which is why *you should pay attention* to both your method and your configuration.

### Checklist

Load-testing should *only* be carried out with Kubernetes. 

Method:

* [ ] I have created a test-plan *with a hypothesis* and *have documented my method* so I can share it with the project team
* [ ] I'm using a performance testing tool such as [hey](https://github.com/rakyll/hey), jMeter, LoadRunner or Gattling
* [ ] My environment is hosted in an isolated and repeatable environment
* [ ] I understand the difference between a benchmark and a "DoS attack"

HA:

* [ ] I have scaled the gateway service in proportion to the load with more than one replica
* [ ] I have set min / max replicas and [understand how auto-scaling works](http://docs.openfaas.com/architecture/autoscaling/)
* [ ] I have read and applied [production-environment recommendations](https://docs.openfaas.com/architecture/production/)

Project tuning:

* [ ] I have extended or removed memory limits / quotas for each service and function
* [ ] I have created my own function using one of the new HTTP templates and the of-watchdog (see below for a list)
* [ ] I understand the difference between the original default watchdog which forks one process per request and the new of-watchdog's HTTP mode and I am using that
* [ ] I have turned off `write_debug` and `read_debug` so that the logs for the function are kept sparse
* [ ] I am monitoring / collecting logs from the core services and function under test
* [ ] I am monitoring the system for feedback through Prometheus and / or Grafana - i.e. throughput and 200/500 errors
* [ ] I am using Kubernetes 1.13 or newer
* [ ] I am not using Docker Swarm
* [ ] ~~If running on Docker Swarm I've verified that I am using a proper HEALTHCHECK (read up more on watchdog readme)~~
* I am using [Endpoint load-balancing](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#endpoint-load-balancing) or [Linkerd2](https://github.com/openfaas-incubator/openfaas-linkerd2)


> A note on DNS: You are likely to get better performance by switching out kube-dns for CoreDNS.

* Watchdog differences

> The [current version of OpenFaaS templates](https://github.com/openfaas/templates) use the original `watchdog` which `forks` processes - a bit like CGI. The newer watchdog [of-watchdog](https://github.com/openfaas-incubator/of-watchdog) is more similar to fastCGI/HTTP and should be used for any benchmarking or performance testing along with one of the newer templates.

[Read more on the differences in the docs](http://docs.openfaas.com/architecture/watchdog/)

of-watchdog templates:

* [Golang HTTP template with the Go stdlib](https://github.com/alexellis/golang-http-template)
* [Node10 HTTP template with Express.js](https://github.com/openfaas-incubator/node10-express-template)
* [Python3 HTTP template with gevent/flask](https://github.com/openfaas-incubator/python-flask-template)

### Common mistakes for performance-testing a project:

* Not communicating intent

Communicate your intent to the project so that we can ensure you haven't missed anything and can share results we have obtained during our own testing. Asking arbitary questions out of context will result in a poor interaction with the community and project.

* Not documenting method and environment

The method and approach should be documented including any important details such as the networking between the test machine and the cluster under test. The Linux version, Kubernetes version, the OpenFaaS component versions, the Docker version and the underlying filesystem being used. The specs of both the test cluster and the test runner including the network overlay driver being used for Kubernetes.

* Using an inappropriate method

There is a difference between performance testing and Denial of Service DoS attacks (i.e. with `siege`). You should use tools which allow a gradual ramp-up and controlled conditions such as [hey](https://github.com/rakyll/hey), [jMeter](https://jmeter.apache.org), [LoadRunner](https://en.wikipedia.org/wiki/LoadRunner) or [Gattling](https://gatling.io).

> See also: [Lab9 - auto-scaling in OpenFaaS](https://github.com/openfaas/workshop/blob/master/lab9.md)

* Choosing an inappropriate test environment

Do not try to performance test OpenFaaS on your laptop within a VM - this carries an overhead of virtualisation and will likely cause contention.

The test environment needs to replicate the production environment you are likely to use. Take note that most AWS virtual machines are subject to CPU throttling and a credits system which will make performance testing hard and unscientific.

* Poor choice of test function

There are several sample functions provided in the project, but that does not automatically qualify them for high-throughput benchmarking or load-testing. It's important to create your own function and understand exactly what is being put into it so you can measure it effectively. You should also use an OpenFaaS of-watchdog template for this or your own microservice conforming to the required healthchecks.

* Not using the of-watchdog

Only the watchdog and of-watchdog implement the correct healthcheck and shutdown signals to be compatible with Kubernetes' startup, scaling and termination mechanisms. If you are running a container exposing port `8080` which does not use of-watchdog then the results are not representative. You will need to mirror the mechanisms in the of-watchdog or use it as a shim.

* Only picking the best/worst case figure

When using a scientific method you need to carry out multiple test runs and account for caching/memory/paging of the operating system including any additional background processes that may be running. The 99th percentile figures should be used, not the best or worst case figure from arbitrary runs.

* Ignoring CPU / memory limits

OpenFaaS enforces memory limits on core services. If you are going to perform a high load test you will want to extend these beyond the defaults or remove them completely.

