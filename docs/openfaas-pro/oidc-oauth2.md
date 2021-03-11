### OIDC and OAuth2 for the OpenFaaS API

You can enable authentication via OpenID Connect and OAuth2 using the OpenFaaS REST API. 
This functionality is part of of the [OpenFaaS Premium Subscription](https://openfaas.com/support/).

* [Get a 14-day free trial here](https://forms.gle/mFmwtoez1obZzm286)

See also: [OpenFaaS and Okta for SSO](https://www.openfaas.com/blog/openfaas-oidc-okta/)

Docs [for the openfaas pro OIDC OAuth2 component can be found here](/openfaas-pro/oidc-oauth2/)

#### Deploy the plugin using the helm chart

You will need two DNS A records and to enable `Ingress` for your Kubernetes cluster. In the example below the sub-zone `oauth.example.com` is used, however you can use a top-level domain or your own sub-zone.

* Gateway - `http://gw.oauth.example.com`
* Auth - `http://auth.oauth.example.com`

Use `arkade` or `helm` and pass the following overrides, or edit your `values.yaml` file:

```sh
export PROVIDER=""              # Set this to "azure" if using Azure AD.
export LICENSE=""               # Obtain a trial from OpenFaaS Ltd, see above for instructions.
export OAUTH_CLIENT_SECRET=""
export OAUTH_CLIENT_ID=""
export DOMAIN="oauth.example.com"

arkade install openfaas \
  --set oauth2Plugin.enabled=true \
  --set oauth2Plugin.provider=$PROVIDER \
  --set oauth2Plugin.license=$LICENSE \
  --set oauth2Plugin.insecureTLS=false \
  --set oauth2Plugin.scopes="openid profile email" \
  --set oauth2Plugin.jwksURL=https://example.eu.auth0.com/.well-known/jwks.json \
  --set oauth2Plugin.tokenURL=https://example.eu.auth0.com/oauth/token \
  --set oauth2Plugin.audience=https://gw.$DOMAIN \
  --set oauth2Plugin.authorizeURL=https://example.eu.auth0.com/authorize \
  --set oauth2Plugin.welcomePageURL=https://gw.$DOMAIN \
  --set oauth2Plugin.cookieDomain=.$DOMAIN \
  --set oauth2Plugin.baseHost=https://auth.$DOMAIN \
  --set oauth2Plugin.clientSecret=$OAUTH_CLIENT_SECRET \
  --set oauth2Plugin.clientID=$OAUTH_CLIENT_ID 
```

The `authorizeURL`, `tokenURL` and `jwksURL` contain my personal tenant URL, remember to customize this to your own from Auth0, or your IDP.

For `cookieDomain` - set the root URL of both of your sub-domains i.e. `.oauth.example.com`, this is so that the cookie set by the auth service can be used by the gateway.

You should also create an additional Ingress and TLS certificate as per below:

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

#### OAuth2 - Access the UI

The UI uses the [code grant flow](https://oauth.net/2/grant-types/authorization-code/).

Visit the gateway and you will be redirected to your IDP to log in: http://gw.oauth.example.com

#### OAuth2 - Access via the CLI (interactive)

The CLI uses the [implicit grant flow](https://oauth.net/2/grant-types/implicit/) for interactive usage such as your daily workflow from your own computer / workstation.

Run the following:

```sh
faas-cli auth \
  --auth-url https://tenant0.eu.auth0.com/authorize \
  --audience http://gw.oauth.example.com \
  --client-id "${OAUTH_CLIENT_ID}"
```

You will receive a token on the command-line and same will be saved to openfaas config file. `faas-cli` will read the token and pass it for future commands which requires authentication. 

You can also export it with `export TOKEN=""` and use it with any command: `faas-cli list --token="${TOKEN}"`

See also: [faas-cli README](https://github.com/openfaas/faas-cli)

#### OAuth2 - Access via the CLI for CI (non-interactive / machine-usage)

Non-inactive or machine-usage is where you need to access the gateway and you cannot follow a web-browser to authenticate. Here, you need to create a special application in your IDP. It will usually be called a "Machine Application" and has a `client_id` and `client_secret`, these are comparable to a username and password.

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
  --audience http://gw.oauth.example.com
```

You will receive a token on the command-line which is also saved in `~/.openfaas/config.yml`.

The `faas-cli` will read the token and pass it for future commands which requires authentication, you can also export it with `export TOKEN=""` and use it with any command: `faas-cli list --token="${TOKEN}"`

See also: [faas-cli README](https://github.com/openfaas/faas-cli/blob/master/README.md)
