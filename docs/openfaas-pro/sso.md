# Single Sign-On

You can enable authentication via OpenID Connect and OAuth2 using the OpenFaaS REST API.

!!! info "Single Sign-On is becoming OpenFaaS IAM"

    The original Single Sign-On feature for OpenFaaS is being rebuilt with fine-grained access control, Web Identity Federation and auditing. [Learn more](/openfaas-pro/iam/overview)

The SSO support has been tested with: Auth0, Okta, Keycloak and Azure Active Directory. If your IdP is ODIC compatible, it should work, but customers can request support using the channels provided to you when you bought your OpenFaaS Pro license.

## Try a walk-through with Okta

The easiest way to try Single Sign-On with OpenFaaS is to follow a complete walk-through. We have one for [Okta here](https://www.openfaas.com/blog/openfaas-oidc-okta/).

## Deploy SSO using the helm chart (advanced)

You will need two DNS A records and to enable `Ingress` for your Kubernetes cluster. In the example below the sub-zone `oauth.example.com` is used, however you can use a top-level domain or your own sub-zone.

* Gateway - `http://gw.oauth.example.com`
* Auth - `http://auth.oauth.example.com`

Create a secret so that the plugin can access your OpenFaaS Pro license:

```bash
kubectl create secret generic \
  -n openfaas \
  openfaas-license \
  --from-file license=$HOME/.openfaas/LICENSE
```

Cookies for UI sessions will be set at the `.oauth.example.com` level so that they can be shared between the gateway and the authentication plugin.

Use `arkade` to install openfaas with the following overrides:

```sh
export PROVIDER=""              # Set this to "azure" if using Azure AD.
export OAUTH_CLIENT_SECRET=""
export OAUTH_CLIENT_ID=""
export DOMAIN="oauth.example.com"

arkade install openfaas \
  --set oidcAuthPlugin.enabled=true \
  --set oidcAuthPlugin.provider=$PROVIDER \
  --set oidcAuthPlugin.insecureTLS=false \
  --set oidcAuthPlugin.scopes="openid profile email" \
  --set oidcAuthPlugin.openidURL=https://example.eu.auth0.com/.well-known/openid-configuration \
  --set oidcAuthPlugin.audience=https://gw.$DOMAIN \
  --set oidcAuthPlugin.welcomePageURL=https://gw.$DOMAIN \
  --set oidcAuthPlugin.cookieDomain=.$DOMAIN \
  --set oidcAuthPlugin.baseHost=https://auth.$DOMAIN \
  --set oidcAuthPlugin.clientSecret=$OAUTH_CLIENT_SECRET \
  --set oidcAuthPlugin.clientID=$OAUTH_CLIENT_ID 
```

> If you prefer, you can also use `helm` and pass the following overrides via `--set key=value`, or edit your `values.yaml` file. But if this is your first time setting up the plugin, you should first try the documentation as given before adapting it to helm. 

> Note: as of version 0.5.0 of the plugin
>
> The `authorizeURL`, `tokenURL` and `jwksURL` values are no longer required. Instead `openidURL` is used to provide the OpenID Discovery endpoint such as `https://example.eu.auth0.com/.well-known/openid-configuration`
> 
> The license is no longer passed as a variable, instead it is accessed via a secret in the `openfaas` namespace.

For `cookieDomain` - set the root URL of both of your sub-domains i.e. `.oauth.example.com`, this is so that the cookie set by the auth service can be used by the gateway.

You should also create an additional Ingress and TLS certificate as per below.

You can use the `openfaas-ingress` arkade app, or create an Ingress record manually.

```bash
  arkade install openfaas-ingress \
  --domain gw.oauth.example \
  --oidc-plugin-domain auth.oauth.example \
  --email webmaster@example.com
```

This is an example of a manual Ingress record created without using arkade.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: openfaas-auth
  namespace: openfaas
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: auth.oauth.example
    http:
      paths:
      - backend:
          serviceName: oauth2-plugin
          servicePort: 8080
        path: /
  tls:
  - hosts:
    - auth.oauth.example
    secretName: openfaas-auth
```

Check that your Ingress is working by visiting: `https://auth.oauth.example.com/health`, make sure you have a valid certificate in place.

### Troubleshooting

Check that your `client_id`, `client_secret`, base host, cookie domain, auth URL, scopes, domains, etc are all set correctly.

You may also run into issues if your redirect domain is not set correctly in the IdP and in your arkade install command.

You can check the logs of the plugin via: `kubectl logs -n openfaas deploy/oauth2-plugin`

OpenFaaS Pro customers have support included as part of their package and can [contact us via email](mailto:contact@openfaas.com).

### Gain access via the UI

The UI uses the [code grant flow](https://oauth.net/2/grant-types/authorization-code/).

Just visit the gateway and you will be redirected to your IdP to log in: http://gw.oauth.example.com/

A cookie will be set in your browser so that you don't have to log in again until it expires.

### Gain access via the CLI (interactive)

For interactive usage such as your daily workflow from your own computer / workstation. The CLI uses the [code grant flow](https://oauth.net/2/grant-types/authorization-code/) [with the PKCE extensions](https://oauth.net/2/pkce/). The `code_challenge_method` for PKCE is set to SHA256, if you require the "plain" method, reach out to support.

Run the following:

```sh
faas-cli auth \
  --grant code \
  --auth-url https://tenant0.eu.auth0.com/authorize \
  --token-url https://tenant0.eu.auth0.com/oauth/token \
  --client-id "${OAUTH_CLIENT_ID}" \
```

> `--audience` is optional

You will receive a token on the command-line and same will be saved to openfaas config file. `faas-cli` will read the token and pass it for future commands which requires authentication.

You can also export it with `export TOKEN=""` and use it with any command: `faas-cli list --token="${TOKEN}"`

> The implicit code grant flow is supported for legacy users, [but is no longer recommended](https://oauth.net/2/grant-types/implicit/). Use the code grant flow with PKCE instead.

See also: [faas-cli README](https://github.com/openfaas/faas-cli)

### Gain access via the CLI for CI (non-interactive / machine-usage)

Non-inactive or machine-usage is where you need to access the gateway and you cannot follow a web-browser to authenticate. Here, you need to create a special application in your IdP. It will usually be called a "Machine Application" and has a `client_id` and `client_secret`, these are comparable to a username and password.

You will need to use the [client credentials flow](https://oauth.net/2/grant-types/client-credentials/).

You will need this flow for any actions taken within a cron-job, broker, CI/CD job or similar server-access.

You can use `faas-cli login`:

```sh
faas-cli login \
 --username ${OAUTH_CLIENT_ID} \
 --password ${OAUTH_CLIENT_SECRET}
```

Now run any command as usual such as `faas-cli list` or `faas-cli deploy`. The secrets will be fetched from `~/.openfaas/config.yml`.

Note: some providers may also support obtaining a token for this flow, such as Auth0:

```sh
faas-cli auth \
  --grant client_credentials \
  --auth-url https://tenant0.eu.auth0.com/oauth/token \
  --client-id "${OAUTH_CLIENT_ID}" \
  --client-secret "${OAUTH_CLIENT_SECRET}"\
  --audience http://gw.oauth.example.com \
```

You will receive a token on the command-line which is also saved in `~/.openfaas/config.yml`.

The `faas-cli` will read the token and pass it for future commands which requires authentication, you can also export it with `export TOKEN=""` and use it with any command: `faas-cli list --token="${TOKEN}"`

See also: [faas-cli README](https://github.com/openfaas/faas-cli/blob/master/README.md)

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
