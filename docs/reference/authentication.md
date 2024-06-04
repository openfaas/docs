# Authentication for functions

There are two main concerns for authentication for OpenFaaS: the administrative API Gateway API and the individual functions.

This page covers authentication for functions, for the OpenFaaS API see: [Docs: OpenFaaS API](/reference/rest-api/).

Functions are exposed on two routes on the OpenFaaS API Gateway:

* `/function/`
* `/async-function/`

Whether your functions require authentication will depend upon the consumers of those functions and the goals of your application. Most OpenFaaS templates enable full control of the HTTP request and response, so you can use any mechanism you like with your functions.

In most cases, whatever you would use with a HTTP server or microservice can be applied with OpenFaaS.

### Built-in authentication

With OpenFaaS Identity and Access Management (IAM) you can secure function endpoints without having to write any additional code. See: [IAM Function Authentication](/openfaas-pro/iam/function-authentication/)

### Webhooks

When functions receive webhooks, most publishers of events will not support any form of authentication. Messages are verified using symmetric keys, where the client (your function) and the server (GitHub, Stripe, YouTube, etc) share the same secret. When a webhook is received, then your function must create a hash of the payload and compare it with a digest sent by the server. If they match, then the message is genuine. This technique is called [HMAC (hash-based message authentication code)](https://en.wikipedia.org/wiki/HMAC).

!!! info "What's HMAC?"
    HMAC involves a shared symmetric secret - both parties store the key securely. The sender computes a hash of the body of the request with their symmetric key and sends this data to the receiver along with the hash value in the HTTP header. The receiver then computes a hash of the body with their copy of the key and checks that this matches what the sender supplied in the HTTP header.

See also: [Lab 11: Enabling trust with HMAC](https://github.com/openfaas/workshop/blob/master/lab11.md) from the OpenFaaS workshop.

A more recent example shows how to validate and verify webhooks from Discord: [Build a serverless Discord bot with OpenFaaS and Golang](https://www.openfaas.com/blog/build-a-serverless-discord-bot/) 

### Websites and portals for users

Websites, portals and dashboards can be built using OpenFaaS.

For authentication and authorization, signing on can be implemented using OAuth and many languages and frameworks have middleware to automate that for you.

As an example of this, see [Alex Ellis' Sponsors Portal](https://insiders.alexellis.io/), it's authenticates users using GitHub and authorizes them if their sponsorship amount covers the minimum amount.

### Bearer tokens and API keys

You can also create and manage your own Bearer tokens or API keys. These are simply shared secrets, where the client (the user or machine invoking the function) and the server (the function) share a copy of the token.

You can assign a token to a function using a secret and then read the value back at run-time to authenticate requests.

Example of invoking a function with a Bearer token:

```
export TOKEN=""

curl https://gw.example.com/function/function1 \
    -H "Authorization: Bearer $TOKEN"
```

See an example written in Python using a HTTP Authorization header: [secrets](/reference/secrets).

### Basic authentication

You can enable basic-authentication for a web-portal or an API by setting the appropriate header such as `WWW-Authenticate: Basic realm="User Visible Realm"`. This is the approach that the OpenFaaS gateway takes, and the OpenFaaS Pro edition comes with an OIDC integration.

Learn more: [Basic auth](https://en.wikipedia.org/wiki/Basic_access_authentication)
