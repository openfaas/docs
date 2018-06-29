# Deployment guide for Docker Swarm

> Note: If you need to use `sudo` to access the `docker` CLI then you should prefix this behind any shell scripts or CLIs used in this guide or any related tutorials.

## 1.0 Initialize Swarm Mode

You can create a single-host Docker Swarm on your laptop with a single command. You don't need any additional software to Docker 17.06 or greater. You can also run these commands on a Linux VM or cloud host.

This is how you initialize your master node:

```bash
$ docker swarm init
```

If you have more than one IP address you may need to pass a string like `--advertise-addr eth0` to this command.

Take a note of the join token

## 1.1 Join any workers you need

Log into your worker node and type in the output from `docker swarm init` on the master. If you've lost this info then type in `docker swarm join-token worker` and then enter that on the worker.

It's also important to pass the `--advertise-addr` string to any hosts which have a public IP address.

!!! warning
    Check whether you need to enable firewall rules for the [Docker Swarm ports listed here](https://docs.docker.com/engine/swarm/swarm-tutorial/).

## 2.0 Deploy the stack

Clone OpenFaaS and then checkout the latest stable release:

```bash
$ git clone https://github.com/openfaas/faas && \
  cd faas && \
  ./deploy_stack.sh
```

!!! note
    If you want to try newer features you can checkout the `master` branch, but we do not recommend that for first-time users.

## 2.1 Test out the UI

Within a few seconds (or minutes if on a poor WiFi connection) the API gateway and associated OpenFaaS images will be pulled into your local Docker library and you will be able to access the UI at:

`http://127.0.0.1:8080`

!!! tip
    If you're running on Linux you may find that accessing `localhost` times out. We recommend using an IPv4 address such as http://127.0.0.1:8080 to prevent any ambiguity.

## 2.2 Deploy the sample functions

The earlier `git clone` included a set of sample functions in `stack.yml`, to deploy them [install the OpenFaaS CLI](/cli/install/) and run:

```
faas deploy
```

## 3.0 Start the hands-on labs

Learn how to build serverless functions with OpenFaaS and Python in our half-day workshop. You can follow along online at your own pace.

* [OpenFaaS workshop](/tutorials/workshop/)

## Troubleshooting

If you are running into any issues please check out the troubleshooting guide and search the documentation / past issues before raising na issue.

* [Troubleshooting guide](https://github.com/openfaas/faas/blob/master/guide/troubleshooting.md)

