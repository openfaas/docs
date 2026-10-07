## Configure Signet

This guide covers how to configure [Signet](/signet/overview/) as an identity provider for OpenFaaS IAM. Signet is a lightweight OpenID Connect (OIDC) provider that can manage local users or delegate login to GitHub.

Before you begin, make sure you have [deployed Signet](/signet/overview). Your Signet instance must have an HTTPS URL reachable from both your browser and the OpenFaaS components.

Install the [Signet CLI](/signet/overview/#option-a-signet-cli) and provision at least one [local user](/signet/overview/#provisioning-identities), or enable [GitHub login](/signet/overview/#federated-login-with-github-optional).

1. Create a client for the OpenFaaS CLI and dashboard

    Set `SIGNET_URL` to your Signet issuer URL and retrieve the management token from the default Helm installation in the `signet` namespace:

    ```sh
    export SIGNET_URL=https://signet.example.com
    export SIGNET_ADMIN_TOKEN=$(kubectl -n signet get secret signet-admin-token \
      -o jsonpath='{.data.admin-token}' | base64 -d)
    ```

    Create a single public client using the Authorization Code flow with Proof Key for Code Exchange (PKCE). Both the CLI and dashboard can use this client without a client secret. Add the callback URLs for the CLI and your dashboard:

    ```sh
    signet client add --public \
      --redirect-url http://127.0.0.1:31111/oauth/callback \
      --redirect-url https://dashboard.openfaas.example.com/auth/callback \
      openfaas
    ```

    Replace the dashboard URL with your dashboard's public URL. If you are only configuring CLI access, omit the dashboard's `--redirect-url` flag.

    For WSL, use `http://localhost:31111/oauth/callback`. See [SSO with faas-cli](/openfaas-pro/sso/cli/#wsl-users).

    When [configuring the dashboard with IAM](/openfaas-pro/dashboard/#configure-the-dashboard-with-iam), set `iam.dashboardIssuer.url` to your Signet issuer URL and `iam.dashboardIssuer.clientId` to `openfaas`. Leave `iam.dashboardIssuer.clientSecret` blank to use PKCE.

2. Register Signet as a trusted issuer with OpenFaaS

    Create a JwtIssuer object in the `openfaas` namespace to register Signet as a trusted issuer for OpenFaaS IAM.

    Example issuer for Signet:

    ```yaml
    apiVersion: iam.openfaas.com/v1
    kind: JwtIssuer
    metadata:
      name: signet
      namespace: openfaas
    spec:
      iss: https://signet.example.com
      aud:
        - openfaas
      tokenExpiry: 12h
    ```

    The `iss` field must match your Signet issuer URL exactly.

    The `aud` field contains the accepted audiences. Signet uses the client ID as the token audience, in this example the Signet client ID is `openfaas`.

    The `tokenExpiry` field sets the expiry time of the OpenFaaS access token.

3. Create Roles and Policies

    Registering a JwtIssuer enables OpenFaaS to trust tokens from Signet. Users also need a matching Role and Policy to access OpenFaaS resources.

    For an example, see the [IAM walkthrough with Signet IdP](/openfaas-pro/iam/signet-idp/).

## SSO with faas-cli

Install [faas-cli and the Pro plugin](/openfaas-pro/sso/cli/), then log in with the public client created above:

```sh
faas-cli pro auth \
  --gateway https://gateway.openfaas.example.com \
  --authority https://signet.example.com \
  --client-id openfaas
```

The CLI uses the Authorization Code flow with PKCE by default. A browser opens so you can log in with a Signet user or, if configured, GitHub. The CLI exchanges the token from Signet for an OpenFaaS access token and saves it for subsequent commands.

See [SSO with faas-cli](/openfaas-pro/sso/cli/) for token renewal and troubleshooting.
