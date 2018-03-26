# Installation

You can install the CLI with a `curl` utility script, `brew` or by downloading the binary from the releases page. Once installed you'll get the `faas-cli` command and `faas` alias.

## Linux/macOS

Utility script with `curl`:

```bash
$ curl -sSL https://cli.openfaas.com | sudo sh
```

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

## Windows

The easiest way to install the faas-cli is through `scoop`:

```powershell
scoop install faas-cli
```

!!! note
    The `scoop` release may not run the latest minor release but is updated regularly.

## Build from source

The [contributing guide](../contributing) has instructions for building from source and for configuring a Golang development environment.