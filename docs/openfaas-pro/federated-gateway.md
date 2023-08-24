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

1. [Setup Keycloak](https://www.keycloak.org/) within your Kubernetes cluster and expose it on the Internet with Ingress, Istio or Inlets.
2. Obtain a client ID and Client Secret for the customer, save these in your Key Management System or encrypt them into your database 
3. Install the federated gateway to the customer's cluster using the [federated-gateway Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/federated-gateway), setting the `issuer` and `audience` in the Helm chart values. The issuer field corresponds to the URL of your Keycloak instance, and the audience is the client ID you created.

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


Here's how to obtain a token via HTTP, you'll need to write similar API calls into your own software, using a HTTP client in your chosen language.

```bash
export IDP_TOKEN_URL=https://keycloak.example.com/realms/openfaas/protocol/openid-connect/token
export CLIENT_ID="fed-gw.example.com"
export CLIENT_SECRET="SUPER_SECURE"

curl -S -L -X POST "${IDP_TOKEN_URL}" \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode "client_id=${CLIENT_ID}" \
--data-urlencode "client_secret=${CLIENT_SECRET}" \
--data-urlencode 'scope=email' \
--data-urlencode 'grant_type=client_credentials'
```

The [go-sdk for OpenFaaS](https://github.com/openfaas/go-sdk) has a code example that you can use to obtain a token from Keycloak, and then make a request to the federated gateway.

```go
func Test_ClientCredentials(t *testing.T) {
    fedGWURL, _ := url.Parse("https://fed-gw.exit.welteki.dev")
	clientID := "fed-gw.exit.welteki.dev"
	clientSecret := "D7lpZQHeplblBo4jKv3SXljTz3pYMhDA"
	tokenURL := "https://keycloak.exit.o6s.io/realms/openfaas/protocol/openid-connect/token"
	scope := "email"
	grantType := "client_credentials"

	auth := NewClientCredentialsTokenSource(clientID, clientSecret, tokenURL, scope, grantType)

	token, err := auth.Token()
	if err != nil {
		t.Fatal(err)
	}

	if token == "" {
		t.Fatal("token is empty")
	}

	client := NewClient(fedGWURL, &ClientCredentialsAuth{tokenSource: auth}, http.DefaultClient)

	fns, err := client.GetFunctions(context.Background(), "openfaas-fn")
	if err != nil {
		t.Fatal(err)
	}

	if len(fns) == 0 {
		t.Fatal("no functions found")
	}

	for _, fn := range fns {
		fnn, err := client.GetFunction(context.Background(), fn.Name, "openfaas-fn")
		if err != nil {
			t.Fatal(err)
		}
		t.Logf("%s - %s - %d/%d", fnn.Name, fnn.Image, fnn.AvailableReplicas, fnn.Replicas)
	}
}
```

Each IdP will have its own way of obtaining tokens, but in general, the path you're looking for will be the "token" URL and is usually found in the "OpenID Endpoint Configuration" page, or via the OpenID Connect Discovery URL. The Discovery URL is often found at `/.well-known/openid-configuration` on the IdP's main URL. In the example of Keycloak, you must also add the `/realms/REALM_NAME` to the path.

Next, whenever you want to communicate with that customer's cluster:

1. Fetch the client ID and Client Secret for that customer.
2. Make a HTTP call to Keycloak's `/token` endpoint along with the client ID and Client Secret
3. Extract the `access_token` variable from the JSON response
4. Make a HTTP call to the federated gateway's REST API with the `access_token` in the `Authorization` header, i.e. `Authorization: Bearer <access_token>`

For testing, you can also use the same token with the OpenFaaS CLI from your laptop, with: `faas-cli list -g https://fed-gw.example.com --token <access_token>`

By default, the federated gateway will only allow the `/system/` endpoints to be accessed, if you'd like to invoke functions over the proxy, set `allowInvoke` to `true` in the Helm chart values.

## FAQ

If you have any questions or comments, please get in touch with us through your usual support channels.

* Where is the Helm chart? It's a separate chart within the [faas-netes repository](https://github.com/openfaas/faas-netes/tree/master/chart/federated-gateway)

* How do you setup Keycloak on Kubernetes? The documentation is rudimentary, but we would advise: applying the main Kubernetes manifest for the Deployment, making sure you've changed the admin password from the default, creating an Ingress record and making sure it gets a TLS certificate

* What is inlets uplink? Inlets Uplink was created by our team to the question: "How do you access customer services from within your own product?" [Read the docs](https://docs.inlets.dev/uplink/overview/)

* How long do tokens last? The default seems to be about 5 minutes, which is configurable in Keycloak.

* Could we use Auth0 or Okta instead? Yes absolutely. Just configure the Client Credentials flow, which is the option that's often used for server to server interaction.

* Why isn't this linked to my Gmail or GitHub Account? The federated gateway was built for server to server communication, the tokens you obtain are not going to be tied to a human identity, which requires a person to be present to login for every API request. This is impractical for server to server communication.

* Can we use this to deploy to our own cluster from GitHub Actions or GitLab CI? No, it does not validate any claims other than the issuer and the audience. This is valid for the use-case of the federated gateway, but if used with a public issuer, where anyone can obtain a token, would mean anyone could access your gateway. See also: [IAM for OpenFaaS](/openfaas-pro/iam/overview)

* Does OpenFaaS support SSO or IAM for the UI and CLI, for human access? Yes absolutely, see: [IAM for OpenFaaS](/openfaas-pro/iam/overview)
