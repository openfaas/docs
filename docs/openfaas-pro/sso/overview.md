Single Sign-On (SSO) for the OpenFaaS CLI and [dashboard](/openfaas-pro/dashboard/) is a feature of [Identity and Access Management (IAM) for OpenFaaS](/openfaas-pro/iam/overview/).

Any OpenID Connect (OIDC) compatible identity provider can be configured and registered with OpenFaaS to provide authentication.

## Create an OIDC App for OpenFaaS

OpenFaaS IAM has been tested with Auth0, Google, Okta, Keycloak and Azure Active Directory but any other provider that supports OIDC should work.

To configure your identity provider:

1. Create a new client (application) for OpenFaaS.
2. Enable the required authentication flows.

    Enable at least one authentication flow for the OpenFaaS CLI and if deployed, the OpenFaaS dashboard.

    Authentication flows supported by the OpenFaaS CLI:

    - Authorization Code flow with Proof Key of Code Exchange (PKCE) - (recommended)
    - Implicit flow
    - Client Credentials flow

    Authentication flows supported by the OpenFaaS dashboard:

    - Authorization Code Flow
    - Authorization Code Flow with Proof Key of Code Exchange (PKCE)

    For the CLI we recommend the Authorization Code flow with PKCE. One of the other flows can be used if your provider does not support PKCE.

3. Configure allowed callback URLs.

    The dashboard uses the callback path: `/auth/callback`. If you are deploying the dashboard allow callbacks on your dashboards public URL, e.g. `https://dashboard.openfaas.example.com/auth/callback`
    
    For the CLI the default callback URL is, `http://127.0.0.1:31111/oauth/callback`

4. Register your provider with OpenFaaS

    Trusted identity providers need to be registered by creating a JwtIssuer object in the OpenFaaS namespace.

    Example JwtIssuer object:

    ```yaml
    ---
    apiVersion: iam.openfaas.com/v1
    kind: JwtIssuer
    metadata:
      name: myprovider.example.com
      namespace: openfaas
    spec:
      iss: https://myprovider.example.com/
      aud:
        - https://gateway.example.openfaas.com/
      tokenExpiry: 12h
    ```

    The `iss` field needs to be set to the url of your provider, eg. `https://accounts.google.com` for google or `https://example.eu.auth0.com/` for Auth0.

    The `aud` field needs to contain a set of accepted audiences. The set is used to validate the audience field in an OIDC access token and verify OpenFaaS is the intended audience. The audience is usually the gateway’s public URL although for some providers it can also be the client id.

    The `tokenExpiry` field can be used to set the expiry time of the OpenFaaS access token.

For more details on how to configure Single Sign-On for OpenFaaS, follow one of our guides or see the documentation for your provider:

- [Keycloak](/openfaas-pro/sso/keycloak/) - Configure Keycloak for OpenFaaS IAM.
- [Google](/openfaas-pro/sso/google/) - Configure Google for OpenFaaS IAM.
- [Microsoft Entra](/openfaas-pro/sso/microsoft-entra) - Configure Microsoft Entra for OpenFaaS IAM.
- [Okta](/openfaas-pro/sso/okta) - Configure Okta for OpenFaaS IAM.

Some providers may need an additional patch or configuration. Feel free to reach out to us.