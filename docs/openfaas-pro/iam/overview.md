## Identity and Access Management (IAM)

OpenFaaS for Enterprises includes a built-in Identity and Access Management (IAM) system that can be used to control access to the OpenFaaS REST API.

* Enable least privilege access to functions and secrets
* Use short-lived tokens for all API operations
* Enable Single Sign-On for all your users
* Deploy from CI/CD systems without long-lived shared secrets
* Audit all operations performed through the OpenFaaS REST API

!!! note
    Classic Single-Sign On (SSO) with OpenID Connect is a stable feature for OpenFaaS for Enterprises.
    
    OpenFaaS IAM is under active development and adds role-based authorization, policies and Web Identity Federation.

See also: [Classic Single Sign On with OpenID Connect](/openfaas-pro/sso)

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

Workflow for machine users:

* If a machine user is authenticating, then an id_token is usually already present.
* It will be exchanged for an OpenFaaS Access token using the OpenFaaS Gateway.

There must be at least one registered OIDC provider for human users to authenticate, and then any number of additional issuers can be defined for Web Identity Federation.

## Permissions

Functions:

* `Function:Read` - List functions, get the status for a function and stream the logs
* `Function:Admin` - Create, update, delete functions

Secrets:

* `Secret:Read` - List secrets
* `Secret:Admin` - Create, update, delete secrets

> Note: (it's not possible to unable to fetch the contents of a secret via API)

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

## FAQ

* What Identity Providers are supported?
  
  Any compliant OpenID Connect provider can be used, such as Auth0, Okta, Keycloak, GitLab, GitHub, etc. Some providers have quirks, which may need an additional patch or configuration. Feel free to reach out to us.

* Which should I use?

  If you'd like to get started quickly, and do not have an OIDC solution with your organisation, you can get started auth [Auth0](https://auth0.com/) for free.

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

* How to I build a multi-tenant system with OpenFaaS IAM?

  For each user or group, create a separate Kubernetes namespace, and then grant them write access to that namespace only.

  There are further restrictions that can be applied such as forcing a non-root user for functions, and Kubernetes limit ranges, and a runtimeClass to sandbox the Pod and its containers from accessing the host.

