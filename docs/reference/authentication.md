# Authentication

There are two main concerns for authentication for OpenFaaS: the administrative API Gateway API and the individual functions.

!!! warning "TLS is not optional"
    You must always enable HTTPS when using OpenFaaS over the Internet, or through an untrusted network, and virtually any network should be considered as untrusted.

## OpenFaaS API

### Basic authentication

When exposing OpenFaaS on the public internet it is important to protect the administrative API endpoints of the API Gateway.

These APIs exist at:

* `/system/`

The OpenFaaS API Gateway provides built-in basic authentication. It is enabled by default for OpenFaaS on Kubernetes and faasd.

Commercial users should explore the [Single Sign-On](/openfaas-pro/sso) feature of OpenFaaS Pro, to prevent the need to share credentials between users and systems.

### Single Sign-On (SSO) and OIDC

Single Sign-On (SSO) uses your identity provider such as Auth0, Okta, Azure Active Directory or GitLab to authenticate users to your OpenFaaS API. It supports the CLI, UI and machine-based authentication. It means that you no longer need to share a single credential with any administrators, and don't need to redeploy your OpenFaaS installation if an employee leaves the company, and you need to rotate the password.

Learn more about (SSO) and OIDC in [OpenFaaS Pro](/openfaas-pro/sso)

### Authentication plugins

When using an auth plugin, the API Gateway will delegate authentication of the `/system/` routes to a microservice.

By default the [basic auth plugin](https://github.com/openfaas/faas/tree/master/auth/basic-auth) is used.

You can configure the gateway to use an auth plugin with the following two environment variables:

* `auth_proxy_url` - the URL to the endpoint to use i.e. `http://auth-module.openfaas:8080/validate`
* `auth_pass_body` - whether to pass the body of the request to the auth module, the default value is `false`

See also: [auth plugins](https://github.com/openfaas/faas/tree/master/auth)

## Authentication for functions

Functions are exposed on two routes on the OpenFaaS API Gateway:

* `/function/`
* `/async-function/`

Whether your functions require authentication will depend upon the consumers of those functions and the goals of your application. Most OpenFaaS templates enable full control of the HTTP request and response, so you can use any mechanism you like with your functions.

In most cases, whatever you would use with a HTTP server or microservice can be applied with OpenFaaS.

### Webhooks

When functions receive webhooks, most publishers of events will not support any form of authentication. Messages are verified using symmetric keys, where the client (your function) and the server (GitHub, Stripe, YouTube, etc) share the same secret. When a webhook is received, then your function must create a hash of the payload and compare it with a digest sent by the server. If they match, then the message is genuine. This technique is called [HMAC (hash-based message authentication code)](https://en.wikipedia.org/wiki/HMAC).

!!! info "What's HMAC?"
    HMAC involves a shared symmetric secret - both parties store the key securely. The sender computes a hash of the body of the request with their symmetric key and sends this data to the receiver along with the hash value in the HTTP header. The receiver then computes a hash of the body with their copy of the key and checks that this matches what the sender supplied in the HTTP header.

See also: [Lab 11: Enabling trust with HMAC](https://github.com/openfaas/workshop/blob/master/lab11.md) from the OpenFaaS workshop.

### Websites and portals for users

Building a portal for users is also a common use-case for functions. You can enable OAuth or a social login by integrating a middleware within your function. Middleware exists for most programming languages to authenticate a function.

### Basic authentication

You can enable basic-authentication for a web-portal or an API by setting the appropriate header such as `WWW-Authenticate: Basic realm="User Visible Realm"`. This is the approach that the OpenFaaS gateway takes, and the OpenFaaS Pro edition comes with an OIDC integration.

Learn more: [Basic auth](https://en.wikipedia.org/wiki/Basic_access_authentication)

### Bearer tokens and API keys

You can also create and manage your own Bearer tokens or API keys. These are simply shared secrets, where the client (the user or machine invoking the function) and the server (the function) share a copy of the token.

You can assign a token to a function using a secret and then read the value back at run-time to authenticate requests.

Example of invoking a function with a Bearer token:

```
curl https://gw.example.com -H "Authorization: Bearer TOKEN"
```

See an example using a HTTP header: [secrets](./secrets.md).
