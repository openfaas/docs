## SSO with faas-cli

For customers who have enabled IAM for OpenFaaS, OIDC can be used to authenticate the CLI. An initial token will be obtained from your Identity Provider (IdP) and then it will be exchanged for an OpenFaaS access token.

You'll need to start off by installing the faas-cli, if you don't have it yet, [it's available on GitHub](https://github.com/openfaas/faas-cli/releases).

!!! note "Do you want to run faas-cli from a server instead of as a human user?"
    SSO and OIDC are primarily designed for interactive use by a human user in front of a keyboard, with a webbrowser available. If you need to use an OAuth token to authenticate server-to-server, then you you'll need to create a new OAuth client in your IdP, and then use the `--client-secret` flag with the `faas-cli pro auth` command. 

### Download the pro plugin

First, download the `pro` plugin which enables SSO using OIDC:

```bash
faas-cli plugin get pro
```

### Enable the plugin

Only authorized use of the plugin is permitted, so you will need to enable it now.

Next, for customers who have a GitHub organisation, you will need to send the OpenFaaS team an email with the name of the organisation, and make sure you can log into your account.

```bash
faas-cli pro enable
```

A browser will open with a device challenge, once completed, the CLI will be enabled.

!!! note "Does your team not have a GitHub organisation available?"
    If your organisation does not use GitHub, or you are not a member of its GitHub organisation, there is an alternative approach to enabling the plugin. Email the OpenFaaS team for the details.

### Log into the gateway using SSO

Now you can log into the gateway using one of the defined JwtIssuers for your installation. If you have not defined a JwtIssuer yet, then see the overview post here: [Walkthrough of Identity and Access Management (IAM) for OpenFaaS](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/).

If your IDP supports Proof Key for Code Exchange PKCE, then you can use the OAuth code flow:

```bash
faas-cli pro auth \
 --client-id CLIENT_ID \
 --authority https://oidc.example.com \
 --gateway https://gateway.example.com
```

A browser will open and you can log in using your IDP.

Following on from that, a JWT will be exchanged for an OpenFaaS access token, and the CLI will be ready to use.

Whenever you want to log in again, you can use the `faas-cli pro auth` command, and you will not need to add the `--client-id` flag or `--authority` flag again, since they will be saved for you.

### Log in to refresh your SSO token

If you have already logged in, then you can refresh your token using the `faas-cli pro auth` command:

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
