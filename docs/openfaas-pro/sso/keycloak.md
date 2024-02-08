## Configure Keycloak

This guide will cover how to configure [Keycloak](https://www.keycloak.org/) as an identity provider for OpenFaaS IAM.

1. Create a new client in your Keycloak realm with Client Type OpenID Connect.

    ![Keycloak client general settings](/images/oidc-configuration/keycloak/keycloak-general-settings.png)

2. Enable the **Standard Flow** (Authorization Code Flow) for the client.

    ![Keycloak client authentication flow settings](/images/oidc-configuration/keycloak/keycloak-auth-flow.png)

3. Add the callback URLs for the CLI and dashboard to the list of valid redirect URIs.

    Add `http://127.0.0.1:31111/oauth/callback` for the CLI. If you are deploying the OpenFaaS dashboard, add the redirect URI for your dashboard e.g `https://dashboard.openfaas.example.com/auth/callback`.

    ![Keycloak client allowed redirect url settings](/images/oidc-configuration/keycloak/keycloak-callback.png)

4. Register your Keycloak provider with OpenFaaS

    Register your Keycloak client as a trusted issuer for OpenFaaS IAM creating a JwtIssuer object in the OpenFaaS namespace.

    Example issuer for a Keycloak provider:

    ```yaml
    ---
    apiVersion: iam.openfaas.com/v1
    kind: JwtIssuer
    metadata:
      name: keycloak.example.com
      namespace: openfaas
    spec:
      iss: https://keycloak.example.com/realms/openfaas
      aud:
        - openfaas
      tokenExpiry: 12h
    ```

    Set the `iss` to the URL of your Keycloak provider.

    The `aud` field needs to contain a set of accepted audiences. For Keycloak this is the client id that was selected in the first step.

    The `tokenExpiry` field can be used to set the expiry time of the OpenFaaS access token.

