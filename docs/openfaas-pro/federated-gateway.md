# Federated Gateway

Manage multiple remote OpenFaaS clusters without having to share and manage long-lived credentials.

!!! info "OpenFaaS Enterprise feature"
    This feature is included for [OpenFaaS Enterprise](/openfaas-pro/introduction) customers.

If your team supports independent OpenFaaS clusters for your customers, then you may benefit from using the federated gateway.

The Federated Gateway is installed within a customer's Kubernetes cluster, and is used to access OpenFaaS instead of the standard gateway. It works with OpenFaaS Standard and OpenFaaS CE.

The problems it aims to solve are primary around sharing credentials.

* You need to obtain the credentials securely for each cluster
* You need to store them each securely
* If the credential changes, you need to update your records
* The admin credential is long lived and may be shared with others

Instead, the federated gateway is a proxy that is deployed to a customer's cluster, and is authenticated with short-lived OpenID Connect (OIDC) tokens instead.

It eliminates all of the above security and administration concerns.

You'll need an OIDC compatible identity provider such as Auth0, Okta, or Azure LDAP. For self-managed installations, Keycloak is a good choice, and quick to get started with.

See also: [Keycloak](https://www.keycloak.org/)

## How it works

Alex gives you a walk-through of how it works in this recording

<iframe width="560" height="315" src="https://www.youtube.com/embed/TPZHa0fw0Yk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

You'll need to add a new client to your OIDC provider for every remote customer cluster.

Alex shows you how to create a new Client within Keycloak, and how to obtain its Client ID and Client Secret.

<iframe width="560" height="315" src="https://www.youtube.com/embed/G2QVhUAEylc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Going end to end

Onboard the customer, do this only once.

1) [Setup Keycloak](https://www.keycloak.org/) within your Kubernetes cluster and expose it on the Internet with Ingress, Istio or Inlets.
2) Obtain a client ID and Client Secret for the customer, save these in your Key Management System or encrypt them into your database 
3) Install the federated gateway to the customer's cluster using the helm chart, setting the `issuer` and `audience` in the Helm chart values. The issuer field corresponds to the URL of your Keycloak instance, and the audience is the client ID you created.

For testing and development, we run Keycloak on a workstation using Docker, and [expose it to the Internet over an inlets HTTPS tunnel](https://docs.inlets.dev/tutorial/automated-http-server/).

Here's the command you can use to run Keycloak locally, if you want to experiment more before setting it up within Kubernetes:

```bash
#!/bin/bash

KEYCLOAK_ADMIN_PASSWORD="SECURE PASSWORD HERE"

docker run \
 -p 8888:8080 \
 -e KEYCLOAK_ADMIN=admin \
 -e KEYCLOAK_ADMIN_PASSWORD=$KEYCLOAK_ADMIN_PASSWORD \
 --name of-iam-keycloak \
 -t quay.io/keycloak/keycloak:21.1.1 start-dev --hostname keycloak.example.com --proxy=edge
```

We recommend accessing the federated gateway over [inlets uplink](https://docs.inlets.dev/uplink/overview/), or a VPN, but you can also expose it on the Internet, if the cluster can obtain a public Load Balancer.

Next, whenever you want to communicate with that customer's cluster:

1) Fetch the client ID and Client Secret for that customer.
2) Make a HTTP call to Keycloak's `/token` endpoint along with the client ID and Client Secret
3) Extract the `access_token` from the JSON response
4) Make a HTTP call to the federated gateway's REST API with the `access_token` in the `Authorization` header, i.e. `Authorization: Bearer <access_token>`

For testing, you can also use the same token with the OpenFaaS CLI from your laptop, with: `faas-cli list -g https://fed-gw.example.com --token <access_token>`

If you have any questions or comments, please get in touch with us through your usual support channels.
