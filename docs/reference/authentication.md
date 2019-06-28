# Authentication for functions

There are two main concerns for authentication for OpenFaaS: the administrative API Gateway API and the individual functions.

## For the API Gateway

### Basic authentication

When exposing OpenFaaS on the public internet it is important to protect the administrative API endpoints of the API Gateway.

These APIs exist at:

* `/system/`

The OpenFaaS API Gateway as of version 0.8.2 provides built-in basic authentication. It is enabled by default for OpenFaaS on Swarm and Kubernetes when using the helm chart. 

It is strongly recommended that you enable basic authentication and use a strong password to protect the `/system/` route. If you prefer to use an alternative authentication strategy then you can also use a reverse proxy such as [Kong](https://getkong.org/docs/) to enable OAuth or another strategy.

The Kubernetes YAML configuration does not use basic authentication at this point and is only useful for quick testing. If you cannot use helm for any reason then use "tillerless" helm to generate YAML files with basic authentication turned on.

Once basic authentication is enabled you will need to use `faas-cli login` before using the CLI.

### Auth plugins

When using an auth plugin, the API Gateway will delegate authentication of the `/system/` routes to a microservice.

By default the [basic auth plugin](https://github.com/openfaas/faas/tree/master/auth/basic-auth) is used.

You can configure the gateway to use an auth plugin with the following two environment variables:

* `auth_proxy_url` - the URL to the endpoint to use i.e. `http://auth-module.openfaas:8080/validate`
* `auth_pass_body` - whether to pass the body of the request to the auth module, the default value is `false`

See also: [auth plugins](https://github.com/openfaas/faas/tree/master/auth)

### OAuth2 support in the API Gateway (alpha)

The OpenFaaS API Gateway has support for OAuth2 and OpenID Connect as of version 0.14.4. This is enabled through the use of an [external authentication module](https://github.com/openfaas/faas/tree/master/auth) as documented above.

You need to use the [odic-plugin](https://github.com/alexellis/oidc-plugin-dist) which is available in binary format for Linux and MacOS on GitHub.

You will need two DNS A records and to enable `Ingress` for your Kubernetes cluster.

* Gateway - `http://gw.example.com`
* Auth - `http://auth.example.com`

You will need to deploy the [oidc-plugin](https://github.com/alexellis/oidc-plugin-dist) provided by OpenFaaS Ltd.

* Deploy using a Kubernetes Deployment, Service and Ingress record [see repo for more](https://github.com/alexellis/oidc-plugin-dist)
* Or deploy as a stand-alone Linux binary using instructions below

Populate the information below according to your Identity Provider (IDP), I'm using Auth0:

```sh
export client_id="your-client-id"                                      
export client_secret="your-secret"  
export cookie_domain=".example.com"
export base_host="http://auth.example.com"
export port=9000
export authorize_url="https://alexellis.eu.auth0.com/authorize"
export welcome_page_url="http://gw.example.com"
export public_key_path="" # leave blank, or populate if JWKS is unavailable
export audience="https://alexellis.eu.auth0.com/api/v2/"
export token_url="https://alexellis.eu.auth0.com/oauth/token"

export scopes="openid profile email read:current_user"
export jwks_url="https://alexellis.eu.auth0.com/.well-known/jwks.json"

./oidc-plugin-linux
```

The `authorize_url` and `jwks_url` contain my personal tenant URL, remember to customise this to your own.

For `cookie_domain` - set the root URL of both of your sub-domains, this is so that the cookie set by the auth service can be used by the gateway.

Edit your gateway configuration i.e. `kubectl edit -n openfaas deploy/gateway` and configure the following:

* `auth_proxy_url` - the URL to the [oidc-plugin](https://github.com/alexellis/oidc-plugin-dist)
* `auth_pass_body` - use the value of `false`

#### OAuth2 - Access the UI

The UI uses the [code grant flow](https://oauth.net/2/grant-types/authorization-code/).

Just visit the gateway and you will be redirected to your IDP to log in: http://gw.example.com

#### OAuth2 - Access via the CLI (interactive)

The CLI uses the [implicit grant flow](https://oauth.net/2/grant-types/implicit/) for interactive usage such as your daily workflow from your own computer / workstation.

Run the following:

```sh
faas-cli auth \
  --auth-url https://tenant0.eu.auth0.com/authorize \
  --audience http://gw.example.com \
  --client-id "${OAUTH_CLIENT_ID}"
```

You will receive a token on the command-line, export it with `export TOKEN=""`.

Then use it with any command: `faas-cli list --token="${TOKEN}"`

See also: [faas-cli README](https://github.com/openfaas/faas-cli)

#### OAuth2 - Access via the CLI (non-interactive / machine-usage)

Non-inactive or machine-usage is where you need to access the gateway and you cannot follow a web-browser to authenticate. Here, you need to create a special application in your IDP. It will usually be called a "Machine Application" and has a `client_id` and `client_secret`, these are comparable to a username and password.

We need to use the [client credentials flow](https://oauth.net/2/grant-types/client-credentials/).

You will need this flow for any actions taken within a cron-job, broker, CI/CD job or similar server-access.

Run the following:

```sh
faas-cli auth \
  --grant client_credentials \
  --auth-url https://tenant0.eu.auth0.com/oauth/token \
  --client-id "${OAUTH_CLIENT_ID}" \
  --client-secret "${OAUTH_CLIENT_SECRET}"\
  --audience http://gw.example.com
```

You will receive a token on the command-line, export it with `export TOKEN=""`.

Then use it with any command: `faas-cli list --token="${TOKEN}"`

See also: [faas-cli README](https://github.com/openfaas/faas-cli)

## For functions

Functions are exposed at:

* `/function/`
* `/async-function/`

Functions exposed on OpenFaaS often do not need to have authentication enabled, this is because they may be responding to webhooks from an external system such as GitHub or Patreon. Neither GitHub, nor Patreon will support authenticating with OAuth or basic authentication strategies, but rely on HMAC.

HMAC involves a shared symmetric secret - both parties store the key securely. The sender computes a hash of the body of the request with their symmetric key and sends this data to the receiver along with the hash value in the HTTP header. The receiver then computes a hash of the body with their copy of the key and checks that this matches what the sender supplied in the HTTP header. See the reference on [secrets](./secrets.md) for a walk-through on using secret values with functions.

See also: [Lab 11: Enabling trust with HMAC](https://github.com/openfaas/workshop/blob/master/lab11.md) from the OpenFaaS workshop.
