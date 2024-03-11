## Identity and Access Management (IAM)

OpenFaaS for Enterprises includes a built-in Identity and Access Management (IAM) system that can be used to control access to the OpenFaaS REST API.

* Enable least privilege access to functions and secrets
* Use short-lived tokens for all API operations
* Enable Single Sign-On for all your users
* Deploy from CI/CD systems without long-lived shared secrets
* Audit all operations performed through the OpenFaaS REST API

## Walkthrough

We recommend you to follow this comprehensive walkthrough blog post to get started with OpenFaaS IAM: [Walkthrough of Identity and Access Management (IAM) for OpenFaaS](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/).

## Workflows

As a user, you can authenticate to OpenFaaS using Open ID Connect with the registered OIDC provider.

* Using `faas-cli pro auth`
* Using the OpenFaaS Pro Dashboard

The registered OIDC provider can also be used for machine to machine access:

* A client ID and client secret can be used with `faas-cli pro auth`

Web Identity Federation support:

* Kubernetes has a built-in OIDC provider, and can be added as a trusted source
* Any accessible OIDC provider such as GitLab, GitHub Actions, Auth0, Okta, Keycloak, can be added as a trusted source

Workflow for human users:

* If a human user is authenticating, then the registered OIDC provider will be used to obtain an id_token.
* This token is then exchanged for an OpenFaaS Access token using the OpenFaaS Gateway.

![Authentication flow for the OpenFaaS CLI](/images/iam/cli-auth-flow.png)
> Authentication flow for the OpenFaaS CLI

Workflow for machine users:

* If a machine user is authenticating, then an id_token is usually already present.
* It will be exchanged for an OpenFaaS Access token using the OpenFaaS Gateway.

![Authentication flow with Identity Federation](/images/iam/ci-auth-flow.png)
> Authentication flow with Identity Federation

There must be at least one registered OIDC provider for human users to authenticate, and then any number of additional issuers can be defined for Web Identity Federation.

## Permissions

Functions:

* `Function:List` - List functions
* `Function:Get` - Get the status of functions
* `Function:Create` - Create functions
* `Function:Update` - Update functions
* `Function:Delete` - Delete functions
* `Function:Logs` - Get logs for functions
* `Function:Scale` - Set the replica count for functions

Secrets:

* `Secret:List` - List secrets
* `Secret:Create` - Create secrets
* `Secret:Update` - Update secrets
* `Secret:Delete` - Delete secrets

> Note: (it's not possible to unable to fetch the contents of a secret via API)

Namespaces:

* `Namespace:List` - List namespaces
* `Namespace:Get` - Get details of namespaces
* `Namespace:Create` - Create namespaces
* `Namespace:Update` - Update namespaces
* `Namespace:Delete` - Delete namespaces

System:

* `System:Info` - Get system provider information

Wildcards can be used to get multiple actions:

* `Function:*` - gives all function permission 
* `*` - gives all permissions for namespaces, functions, secrets etc

Scope of access:

Permissions can be scoped cluster wide, or to a specific namespace:

* `*` - cluster-wide access
* `staging:*` - access to the `staging` namespace only

## Concepts

OpenFaaS IAM objects are defined in the `openfaas` namespace, and need to be created by a system administrator using kubectl, Helm or a GitOps tool.

* JwtIssuer - defines a trusted issuer for OpenFaaS IAM
* Policy - defines a set of permissions and objects on which they can be performed
* Role - defines a set of policies that can be matched to a particular user or identity

## Examples

* [User authentication and IAM with Auth0](/openfaas-pro/iam/example-auth0)

### Web Identity Federation

Web Identity Federation allows you to build a trust relationship with an external identity provider, without sharing long-lived secrets.

* [Deployment from GitHub Actions](/openfaas-pro/iam/github-federation)
* [Deployment from GitLab.com](/openfaas-pro/iam/gitlab-federation)

## OpenFaaS JWT token

During authentication an OIDC access token from a trusted IdP is exchanged for an OpenFaaS access token

The faas-cli can be used to get an access token from your IdP and print it out for inspection. This can be useful to see what fields are available when creating Roles.

To print out the decoded OpenFaaS JWT token run:

```sh
faas-cli pro auth \
  --gateway https://gateway.example.com \
  --authority https://keycloak.example.com/realms/openfaas \
  --client-id openfaas \
  --pretty
```

Adding the `--no-exchange` flag stops the CLI from exchanging the access token for an OpenFaaS token. It can be used to print out the original JWT token issued by your IdP for inspection.

Example of a decoded OpenFaaS JWT token:

```json
{
  "header": {
    "alg": "ES256",
    "kid": "ARyHBtoRuXRzCEqR_9NRr_HPP_s36vHKf9_X_Mjpad4x",
    "typ": "JWT"
  },
  "payload": {
    "at_hash": "F53gv7injmJ9hUGCOtDgBw",
    "aud": "https://gateway.example.com",
    "auth_time": 1706088607,
    "azp": "openfaas",
    "email_verified": true,
    "exp": 1706131945,
    "family_name": "Verstraete",
    "fed:iss": "https://keycloak.example.com/realms/openfaas",
    "given_name": "Han",
    "groups": [
      "openfaas-dev"
    ],
    "iat": 1706088745,
    "iss": "https://gateway.example.com",
    "jti": "81d42f43-8ebe-4d62-84f8-e1ecab5eb4b1",
    "name": "Han Verstraete",
    "nonce": "1706088744941521000",
    "policy": [
      "fn-rw"
    ],
    "preferred_username": "welteki",
    "session_state": "65c3df72-dad8-4b2f-95aa-f1556b40ab80",
    "sid": "65c3df72-dad8-4b2f-95aa-f1556b40ab80",
    "sub": "fed:a81bcb85-72a8-446a-9263-004944a4e9f4",
    "typ": "ID"
  }
}
```

The OpenFaaS JWT contains all claims from the original access token and some additional fields.

* `fed:iss` - The federated issuer that issued the original JWT token.
* `policy` - The list of policies that are mapped to the access token.


## Custom TLS Certificate Authority bundle

Add a certificate bundle to OpenFaaS components for use with an internal certificate authority or self signed certificates

Create a secret that contains the CA bundle in the OpenFaaS namespace:

```
kubectl create secret generic \
  -n openfaas \
  ca-bundle \
  --from-file=ca.crt=ca.crt
```

Update the OpenFaaS chart and add a reference to the Kubernetes secret with the CA bundle:

```yaml
caBundleSecretName: ca-bundle
```

## FAQ

* What Identity Providers are supported?
  
    Any compliant OpenID Connect provider can be used, such as Auth0, Okta, Keycloak, GitLab, GitHub, etc. Some providers have quirks, which may need an additional patch or configuration. Feel free to reach out to us.

* Which should I use?

    If you'd like to get started quickly, and do not have an OIDC solution with your organisation, you can get started with [Auth0](https://auth0.com/) on one of their free plans.

* How can I integrate LDAP users to OpenFaaS IAM?

    OpenFaaS IAM is designed to work with any OIDC provider, so you can use any OIDC provider that supports LDAP integration. Examples include Auth0, Azure Active Directory, Okta, Keycloak or [Dex](https://github.com/dexidp/dex).

* What is the difference between OpenFaaS IAM and Classic OpenFaaS SSO?

    OpenFaaS SSO is a stable feature that covers authentication only. It can be made to further restrict authentication to certain users or emails by creating rules within the OIDC provider itself.

    OpenFaaS IAM is a complete authentication, authorization and Web Identity Federation solution.

* What is the difference between OpenFaaS IAM and an admin password?

    The default for OpenFaaS CE is to use a single administrative password. It is long lived, and rarely changed, so it's not advisable to share this with your team or to encode it within your CI/CD systems.

    OpenFaaS Pro users should use Classic SSO to prevent password sharing and to increase security.

* Do I need to set up OpenFaaS IAM in development or staging, as well as production?

    We recommend using basic authentication in developer environments, and then setting up OpenFaaS IAM in staging and production.

    In staging, you could grant broader permissions to your team, and then restrict permissions in production.

* How do I build a multi-tenant system with OpenFaaS IAM?

    There are many considerations, and we'd encourage you to contact the support team to discuss your requirements.

    As a guide, for each user or group, create a separate Kubernetes namespace, and then grant them write access to that namespace only.

    There are further restrictions that can be applied such as forcing a non-root user for functions, and Kubernetes limit ranges, and a runtimeClass to sandbox the Pod and its containers from accessing the host.

    If you're using Google Compute Engine, you can create a node pool with gVisor enabled with a runtimeClass. For Amazon Elastic Kubernetes Service, Fargate can be used for isolation using a runtimeClass and node pool.

* Can we use self-signed certificates or our own private Certificate Authority?

    Yes, you can use your own private CA, or self-signed certificates. Create a secret called `ca-bundle` in the `openfaas` namespace following the [exact instructions in the Helm chart README file](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values.yaml), then set the `caBundleSecretName:` field in the `values.yaml` file to `ca-bundle`.
