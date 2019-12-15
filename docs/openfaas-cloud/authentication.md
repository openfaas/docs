## OpenFaaS Cloud Auth

OpenFaaS Cloud offers authentication and authorization to the OpenFaaS Cloud Dashboard through OAuth2 / OIDC.

Two providers are supported:

* GitHub.com
* GitLab self-hosted

### Conceptual diagram

OpenFaaS Authentication with OAuth2 / OIDC

[![Conceptual diagram](/images/openfaas-cloud/oauth2.png)](/images/openfaas-cloud/oauth2.png)

### Control-plane

The control-plane for OpenFaaS Cloud is federated to the source-control management system through the use of webhooks, validated with a symmetric secret. Users do not have access to the OpenFaaS REST API other than through causing events to trigger through Git.

See also: [OpenFaaS Cloud architecture](../architecture/)
