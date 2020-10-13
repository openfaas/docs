# Deployment guide for Docker Swarm

> Note: If you need to use `sudo` to access the `docker` CLI then you should prefix this behind any shell scripts or CLIs used in this guide or any related tutorials.

## 1.0 Install the `faas-cli`

You can install the OpenFaaS CLI using `brew` or a `curl` script.

* via `brew`:

```bash
brew install faas-cli
```

* via `curl`:

```bash
$ curl -sL https://cli.openfaas.com | sudo sh
```

If you run the script as a normal non-root user then the script will be downloaded to the current folder.

## 1.1 Initialize Swarm Mode

You can create a single-host Docker Swarm on your laptop with a single command. You don't need any additional software to Docker 17.06 or greater. You can also run these commands on a Linux VM or cloud host.

This is how you initialize your master node:

```bash
$ docker swarm init
```

If you have more than one IP address you may need to pass a string like `--advertise-addr eth0` to this command.

Take a note of the join token

## 1.2 Join any workers you need

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

!!! info
    As of OpenFaaS 0.8.6 basic authentication will be enabled by default when running ./deploy\_stack.sh. If you need to disable it pass the flag `--no-auth` to the ./deploy_stack.sh script as above.

### 2.0.1 Raspberry Pi / 32-bit ARM (armhf)

> For a complete tutorial on setting up OpenFaaS for Raspberry Pi / 32-bit ARM using Docker Swarm see the following blog post from Alex Ellis: [Your Serverless Raspberry Pi cluster with Docker](https://blog.alexellis.io/your-serverless-raspberry-pi-cluster/).

When creating new functions you will need to run the build on an armhf host.

> Note: expert users can create or use [multi-arch templates](https://github.com/alexellis/multiarch-templates) which can build on a PC and deploy to an armhf host.

* You can run `faas-cli deploy` from any computer using `--gateway` or `OPENFAAS_GATEWAY`
* But you must build Docker images on a Raspberry Pi, not on your PC or laptop. 

For the Function Store, use the following:

```bash
faas-cli store list --platform armhf
faas-cli store deploy NAME
```

### 2.1 Store your admin credentials

The default configuration will create a username and password combination for you:

```
Attempting to create credentials for gateway..
...
[Credentials]
 username: admin
 password: <some_hash_secret>
 echo -n <some_hash_secret> | faas-cli login --username=admin --
password-stdin
```

Run the command as you see it in your console, do not copy/paste the login command. Once you run `faas-cli login` your password will be stored as a hash at `~/.openfaas/config.yml`.

You will need the password for the CLI, UI and REST API on the gateway, but you can invoke your functions without it.

## 2.1 Test out the UI

Within a few seconds (or minutes if on a poor WiFi connection) the API gateway and associated OpenFaaS images will be pulled into your local Docker library and you will be able to access the UI at:

`http://127.0.0.1:8080`

!!! tip
    If you're running on Linux you may find that accessing `localhost` times out, most probably because it resolves to IPv6, and (IPv6 is not supported by docker-compose v3 and swarm)[https://docs.docker.com/compose/compose-file/#enable_ipv6]. We recommend using an IPv4 address such as http://127.0.0.1:8080 to prevent any ambiguity.

## 2.2 Deploy functions from the OpenFaaS Function Store

You can find many different sample functions from the community through the OpenFaaS Function Store. The Function Store is built into the UI portal and also available via the CLI.

To search the store:

```bash
$ faas-cli store list
```

To deploy `figlet`:

```bash
$ faas-cli store deploy figlet
```

Now find the function deployed in the cluster and invoke it.

```bash
$ faas-cli list
$ echo "OpenFaaS!" | faas-cli invoke figlet
```

You can also access the Function Store from the Portal UI and find a range of functions covering everything from machine-learning to network tools.

## 3.0 Start the hands-on labs

Learn how to build serverless functions with OpenFaaS and Python in our half-day workshop. You can follow along online at your own pace.

* [OpenFaaS workshop](/tutorials/workshop/)

## Troubleshooting

If you are running into any issues please check out the troubleshooting guide and search the documentation / past issues before raising na issue.

* [Troubleshooting guide](/deployment/troubleshooting/)

