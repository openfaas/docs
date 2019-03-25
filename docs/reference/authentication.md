# Authentication for functions

There are two main concerns for authentication for OpenFaaS: the administrative API Gateway API and the individual functions.

## For the API Gateway

When exposing OpenFaaS on the public internet it is important to protect the administrative API endpoints of the API Gateway.

These APIs exist at:

* `/system/`

It is strongly recommended that you enable basic authentication and use a strong password to protect the `/system/` route. If you prefer to use an alternative authentication strategy then you can also use a reverse proxy such as [Kong](https://getkong.org/docs/) to enable OAuth or another strategy.

The OpenFaaS API Gateway as of version 0.8.2 provides built-in basic authentication. It is enabled by default for OpenFaaS on Swarm and Kubernetes when using the helm chart. 

The Kubernetes YAML configuration does not use basic authentication at this point and is only useful for quick testing. If you cannot use helm for any reason then use "tillerless" helm to generate YAML files with basic authentication turned on.

Once basic authentication is enabled you will need to use `faas-cli login` before using the CLI.

## For functions

Functions are exposed at:

* `/function/`
* `/async-function/`

Functions exposed on OpenFaaS often do not need to have authentication enabled, this is because they may be responding to webhooks from an external system such as GitHub or Patreon. Neither GitHub, nor Patreon will support authenticating with OAuth or basic authentication strategies, but rely on HMAC.

HMAC involves a shared symmetric secret - both parties store the key securely. The sender computes a hash of the body of the request with their symmetric key and sends this data to the receiver along with the hash value in the HTTP header. The receiver then computes a hash of the body with their copy of the key and checks that this matches what the sender supplied in the HTTP header. See the reference on [secrets](./secrets.md) for a walk-through on using secret values with functions.

See also: [Lab 11: Enabling trust with HMAC](https://github.com/openfaas/workshop/blob/master/lab11.md) from the OpenFaaS workshop.
