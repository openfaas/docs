# Installation

We recommend installing faas-cli through arkade, for the quickest and easiest installation.

There is also a bash script for downloading the binary, and support for `brew`.

## Installation with arkade

Install arkade:

```bash
curl -sSL https://get.arkade.dev | sudo -E sh
```

Install faas-cli:

```bash
arkade get faas-cli
```

You can update faas-cli at any time by running the above command again.

arkade can also be used to download around 120 different DevOps and Kubernetes tools, [find out more in the arkade README](https://arkade.dev)

## Installation with Bash

For Linux or macOS users, the following script can be used:

```bash
curl -sSL https://cli.openfaas.com | sudo -E sh
```

> The flag `-E` allows for any `http_proxy` environmental variables to be passed through to the installation bash script.

Non-root with curl downloads the binary into your current directory and will then print installation instructions:

```bash
curl -sSL https://cli.openfaas.com | sh
```

Via brew:

```bash
brew install faas-cli
```

!!! note
    The version of faas-cli installed by `brew` is likely to lag behind a little, so we recommend using arkade instead.

## Windows

In PowerShell:

```powershell
$version = (Invoke-WebRequest "https://api.github.com/repos/openfaas/faas-cli/releases/latest" | ConvertFrom-Json)[0].tag_name
(New-Object System.Net.WebClient).DownloadFile("https://github.com/openfaas/faas-cli/releases/download/$version/faas-cli.exe", "faas-cli.exe")
```

## Environment variable overrides

Several overrides exist which will be used by default if set and no other command-line flag has been set.

* `OPENFAAS_TEMPLATE_URL` - to set the default URL to pull templates from
* `OPENFAAS_PREFIX` - for use with `faas-cli new` - this can act in place of `--prefix`
* `OPENFAAS_URL` - to override the default gateway URL

## Running `faas-cli` with sudo

If you're running the faas-cli with `sudo` we recommend using `sudo -E` to pass through any environmental variables you may have configured such as a `http_proxy`, `https_proxy` or `no_proxy` entry.

## Running `faas-cli` with docker

The `faas-cli` is also available as a Docker image making it convenient for use in CI jobs such as with a Jenkins pipeline or a task in cron.

![ghcr.io/openfaas/faas-cli](https://ghcr.io/openfaas/faas-cli)

There is no "latest" tag, so find the version of the CLI you want to use from the tags page on the Docker Hub. These correspond to the release from GitHub.

> Note: the Docker image cannot be used to perform a build directly, but you can use it to generate a build context which can be used with a container builder such as the OpenFaaS Pro Function Builder API, Docker, buildkit or Kaniko in another part of your build pipeline.

Use-cases for the Docker image:

* Generate the build context without running docker with the `faas-cli build --shrinkwrap` command
* Deploy an existing image to a remote server `faas-cli deploy`
* Manage secrets with `faas-cli secret`
* Invoke functions via cron with `faas-cli invoke`
* Check the health of your remote gateway with `faas-cli info`

## Building from source

The [contributing guide](/community/#contribute) has instructions for building from source and for configuring a Golang development environment. 

* **Star/fork** on GitHub: [faas-cli](https://github.com/openfaas/faas-cli)

## Tutorial

[5 tips and tricks for the OpenFaaS CLI by Alex Ellis](https://www.openfaas.com/blog/five-cli-tips/)
