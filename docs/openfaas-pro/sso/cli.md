## SSO with faas-cli

For customers who have enabled IAM for OpenFaaS, OIDC can be used to authenticate the CLI. An initial token will be obtained from your Identity Provider (IdP) and then it will be exchanged for an OpenFaaS access token.

You'll need to start off by installing the faas-cli and the pro plugin. See: [CLI Installation](/cli/install) and [Pro plugin](/cli/install/#pro-plugin).

!!! note "Do you want to run faas-cli from a server instead of as a human user?"

    SSO and OIDC are primarily designed for interactive use by a human user in front of a keyboard, with a web-browser available. If you need to use an OAuth token to authenticate server-to-server, then you you'll need to create a new OAuth client in your IdP, and then use the `--client-secret` flag with the `faas-cli pro auth` command. 

### Log into the gateway using SSO

Now you can log into the gateway using one of the defined JwtIssuers for your installation. If you have not defined a JwtIssuer yet, then see the overview post here: [Walkthrough of Identity and Access Management (IAM) for OpenFaaS](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/).

The faas-cli supports several OAuth 2.0 authorization flows for obtaining a token from your IdP:

* `device_code` - Device authorization flow. Intended for authenticating a CLI on headless or remote systems.
* `code` - Authorization code flow with PKCE. Interactive login that opens a browser on the machine running the CLI.
* `client_credentials` - Non-interactive, server-to-server authentication using a client ID and secret.
* `implicit` / `implicit-id` - Legacy implicit flows, kept for compatibility with older IdPs.

Select which flow to use by passing its name to the `--grant` flag of the `faas-cli pro auth` command.

#### Device authorization grant

The device authorization flow lets you authenticate without a browser on the machine running the CLI. It prints a verification URL and a one-time code that you open from a browser on any other device to complete the login. This makes it well suited to headless or remote machines, such as a server accessed over SSH.

```bash
faas-cli pro auth \
  --grant device_code \
  --client-id CLIENT_ID \
  --authority https://oidc.example.com \
  --gateway https://gateway.example.com
```

Open the printed verification URL in a browser on any device, enter the code, and complete the login with your IdP. Once you have authorized the request, the JWT is exchanged for an OpenFaaS access token, and the CLI is ready to use.

If the machine running the CLI has a browser available, it will be opened on the verification page for you automatically. To disable this and only print the URL and code, add `--launch-browser=false`:

```bash
faas-cli pro auth \
  --grant device_code \
  --client-id CLIENT_ID \
  --authority https://oidc.example.com \
  --gateway https://gateway.example.com \
  --launch-browser=false
```

!!! note "Your IdP must support the device authorization flow"

    The device authorization flow can only be used when your Identity Provider supports it. See the [SSO overview](/openfaas-pro/sso/overview/) for the supported providers and flows. If your provider does not support it, use the OAuth code flow described below instead.

#### Authorization code flow with PKCE

With the authorization code flow with Proof Key for Code Exchange (PKCE), the CLI opens a browser to sign in and starts a local server to receive the token on the callback URL `http://127.0.0.1:31111/oauth/callback`. This URL needs to be registered as a valid redirect URI on your IdP client.

```bash
faas-cli pro auth \
 --client-id CLIENT_ID \
 --authority https://oidc.example.com \
 --gateway https://gateway.example.com
```

After receiving the token from your IdP, the CLI exchanges it for an OpenFaaS access token and is ready to use.

### Log in to refresh your SSO token

If you have already logged in, then you can refresh your token using the `faas-cli pro auth` command.  Your settings are saved after the first login, so you will not need to provide all of the flags again.

```bash
faas-cli pro auth \
  --gateway https://gateway.example.com
```

### Troubleshooting

#### WSL users

In order to launch a web browser from WSL, you'll need to install the `wslu` package, which is preinstalled with Ubuntu 22.04 and later.

```bash
sudo apt update -qy && sudo apt install -qy wslu --no-install-recommends
```

Then, whenever a browser is launched, it will open on your Windows host, and the result will come back to the WSL environment.

!!! note "Note for Windows Subsystem for Linux (WSL) users"
        When authenticating from WSL, the redirect address will be changed from `http://127.0.0.1` to `http://localhost`. Make sure that the address: `http://localhost:31111/oauth/callback` is added as a callback URL for the IdP app for OpenFaaS.

#### View the token for debugging purposes

To view the token from your IdP, run:

```bash
faas-cli pro auth \
    --gateway https://gateway.example.com \
    --authority https://oidc.example.com \
    --print-token \
    --pretty \
    --no-exchange
```

To view the OpenFaaS access token which was produced in exchange for the token from your IdP, run:

```bash
faas-cli pro auth \
    --gateway https://gateway.example.com \
    --authority https://oidc.example.com \
    --print-token \
    --pretty
```

### For anything else

Contact the OpenFaaS team for additional help and support.
