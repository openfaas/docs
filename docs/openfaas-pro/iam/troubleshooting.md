# Troubleshooting Identity and Access Management (IAM)

Troubleshoot issues with OpenFaaS Identity and Access Management (IAM).

## Unable to access the dashboard

### Authentication fails for the dashboard

Verify the dashboard is configured to support Single Sign-On (SSO). See: [Configure the dashboard with IAM](/openfaas-pro/dashboard/#configure-the-dashboard-with-iam).

The dashboard logs can show why the authentication process with your identity provider failed.

```bash
kubectl logs -n openfaas deploy/dashboard
```

### A user is not authorized to access the dashboard

A user is able to authenticate but the dashboard displays the `Not Authorized` message after logging in.

![Not authorized page](/images/dashboard/not-authorized.png)

This message is displayed when an access token was obtained successfully from your identity provider but the dashboard was unable to exchange it for an OpenFaaS access token. The reason the token exchange failed is can be found in the dashboard logs:

```bash
kubectl logs -n openfaas deploy/dashboard
```

Look for the log messages, `Failed to exchange ID Token for an OpenFaaS ID token`, e.g.

```
2024-02-20T14:45:47.594Z        error   handlers/oauth_callback.go:61   Failed to exchange ID Token for an OpenFaaS ID token    {"error": "cannot fetch token: 400 Bad Request\nResponse: {\"error\":\"invalid_request\",\"error_description\":\"No policies found for subject: a81bcb85-72a8-446a-9263-004944a4e9f4\"}"}
```

The most common reasons the token exchange fails:

1. You don't have any IAM Roles and Policies applied in your cluster yet.

2. There are no matching Roles or Policies for the authenticated user.

3. The identity provider that issued the initial access token is not registered as a trusted issuer for OpenFaaS.

See: [Failed to exchange the ID token for an OpenFaaS token](#failed-to-exchange-the-id-token-for-an-openfaas-token)

## Failed to exchange the ID token for an OpenFaaS token

Some of the most common reasons the token exchange fails.

### No matching roles or policies

You don't have any IAM Roles and Policies applied in your cluster yet or no matching roles were found for the authenticated user.

### Could not validate JWT token, JWKS keys missing.

This error occurs when the oidc-plugin does not have valid JWKS keys cached for an identity provider.

This can happen when the cache gets out of sync or when the plugin is unable to fetch the JWKS keys for a JWTIssuer.

1. Restart the oidc-plugin to force a resync of the cache.

  ```bash
  kubectl rollout restart deploy/oidcs-plugin -n openfaas
  ```

2. Check the logs of the oidc-provider to see if there are any errors while fetching the JWKS keys:

  ```bash
  kubectl logs -n openfaas deploy/oidc-plugin
  ```

### Issuer not registered with OpenFaaS

The identity provider that issued the initial access token is not registered as a trusted issuer for OpenFaaS.

Check your identity provider is registered as a JWTIssuer:

```bash
kubectl get jwtissuer -n openfaas
```

To register your provider, see: [Single Sign-On (SSO) for the OpenFaaS ](/openfaas-pro/sso/overview/)

### Audience not valid

OpenFaaS is not the intended audience of the token.

Check if the token issued by your identity provider has the correct audience. Ensure the audience is in the list of accepted audiences in the JWTIssuer object for your identity provider.

!!! tip "Inspect the IdP token for debugging purposes"

    Use the OpenFaaS CLI to [inspect the token](/openfaas-pro/sso/cli/#view-the-token-for-debugging-purposes) from your IdP.

The `aud` field needs to contain a set of accepted audiences. The audience is usually the gateway’s public URL although for some providers it can also be the client id.

See step 4 in, [Register your provider with OpenFaaS](/openfaas-pro/sso/overview/#create-an-oidc-app-for-openfaas) for reference.

## Certificate signed by unknown authority

The dashboard or oidc-plugin fails to start with an error `x509: certificate signed by unknown authority`. This can happen when the TLS for the gateway or your identity provider uses an internal certificate authority or self signed certificates.

A custom CA bundle can be added through the helm chart. See: [Custom CA bundle for OpenFaaS IAM](/openfaas-pro/iam/overview/#custom-tls-certificate-authority-bundle)