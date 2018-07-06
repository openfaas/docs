# Installation

You can install the CLI with a `curl` utility script, `brew` or by downloading the binary from the releases page. Once installed you'll get the `faas-cli` command and `faas` alias.

## Linux/macOS

Utility script with `curl`:

```bash
$ curl -sSL https://cli.openfaas.com | sudo -E sh
```

> The flag `-E` allows for any `http_proxy` environmental variables to be passed through to the installation bash script.

Non-root with curl (requires further actions as advised after downloading):

```bash
$ curl -sSL https://cli.openfaas.com | sh
```

Via brew:

```bash
$ brew install faas-cli
```

!!! note
    The `brew` release may not run the latest minor release but is updated regularly.

In PowerShell:

```powershell
$version = (Invoke-WebRequest "https://api.github.com/repos/openfaas/faas-cli/releases/latest" | ConvertFrom-Json)[0].tag_name
(New-Object System.Net.WebClient).DownloadFile("https://github.com/openfaas/faas-cli/releases/download/$version/faas-cli.exe", "faas-cli.exe")
```

## Running `faas-cli` with sudo

If you're running the faas-cli with `sudo` we recommend using `sudo -E` to pass through any environmental variables you may have configured such as a `http_proxy`, `https_proxy` or `no_proxy` entry.

## Docker image

The `faas-cli` is also available as a Docker image making it convenient for use in CI jobs such as with a Jenkins pipeline or a task in cron.

https://hub.docker.com/r/openfaas/faas-cli/tags/

There is no "latest" tag, so find the version of the CLI you want to use from the tags page on the Docker Hub. These correspond to the release from GitHub.

## Build from source

The [contributing guide](../contributing) has instructions for building from source and for configuring a Golang development environment.

## Tutorial: learn how to use the CLI

[Morning coffee with the OpenFaaS CLI](https://blog.alexellis.io/quickstart-openfaas-cli/)
