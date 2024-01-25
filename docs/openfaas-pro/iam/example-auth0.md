# Auth0 Example for OpenFaaS IAM

In order to access the OpenFaaS API, a JWT Issuer must first be registered with the system.

Create an application on Auth0 for the OpenFaaS gateway, you'll need to obtain the corresponding "client_id".

## Register the Issuer for Auth0

An Issuer for `https://alexellis.eu.auth0.com/` might look like this:

```yaml
apiVersion: iam.openfaas.com/v1
kind: JwtIssuer
metadata:
  name: alexellis.eu.auth0.com
  namespace: openfaas
spec:
  iss: https://alexellis.eu.auth0.com/
  aud:
    - 17F3M3rS8ORQUPDHsgkq0YVHheZVH8dpaGHRTjAx5x0
    - MO7Eq6O53SOxr3ie19TUMvo71ioYouJHsJEIw0PHc
  tokenExpiry: 12h
```

## Define a Role

Once registered, a Role must be created which maps users within the Issuer to be mapped to a set of Policies

```yaml
apiVersion: iam.openfaas.com/v1
kind: Role
metadata:
  name: dev-staff-deployers
  namespace: openfaas
spec:
  policy:
  - dev-rw
  - staging-readonly
  principal:
    jwt:sub:
      - github|1234567
      - github|7654321
  condition:
    StringEqual:
      jwt:iss: ["https://alexellis.eu.auth0.com/"]
```
> A Role including statements to evaluate its bindings to: two staff members

Valid conditions include: `StringEqual` or `StringLike`.

Every condition must return true for the Role to be considered as a match.

The principal field is optional, however if it is given, both the principal and the condition must match. If there are multiple items given, then only one must match the token.

If you configure Auth0 to emit a "group" claim such as "example.com/group", you could match this with a condition, instead of specifying individual "sub" fields.

A user's email could also be fuzzy matched with a condition, for example:

```yaml
  condition:
    StringLike:
      jwt:email: ["*@example.com"]
```

## Bind a Policy to a Role

Finally, one or more Policies must be created which describe which permissions a user has, and on which resources.

```yaml
apiVersion: iam.openfaas.com/v1
kind: Policy
metadata:
  name: dev-rw
  namespace: openfaas
spec:
  statement:
  - sid: 1-rw-dev
    action:
    - Function:List
    - Function:Get
    - Function:Create
    - Function:Update
    - Function:Delete
    - Function:Logs
    - Function:Admin
    - Secret:List
    - Secret:Create
    - Secret:Update
    - Secret:Delete
    effect: Allow
    resource: ["dev:*"]
```

> Allow read and write to functions and secrets within the `dev` namespace:

```yaml
apiVersion: iam.openfaas.com/v1
kind: Policy
metadata:
  name: staging-readonly
  namespace: openfaas
spec:
  statement:
  - sid: 1-ro-staging
    action:
    - Function:Read
    effect: Allow
    resource:
    - "staging-fn:*"
```

> Allow only read access to functions within the `staging-fn` namespace:

The JwtIssuer, Role and Policy resources are Kubernetes Custom Resources, and must be created within the `openfaas` namespace.

## Authenticate as the user

The `faas-cli` needs to be used to obtain a token from Auth0, and then exchange it for an OpenFaaS Access token.

```bash
faas-cli pro auth \
  --gateway https://gateway.example.com \
  --grant code \
  --authority https://example.eu.auth0/ \
  --client-id 17F3M3rS8ORQUPDHsgkq0YVHheZVH8dpaGHRTjAx5x0
```

The faas-cli will save the OpenFaaS Access token and use it when you run commands that require authentication to the gateway.

Running the following command will list functions in the `dev` namespace of the authenticated user has sufficient permissions.
```bash
faas-cli list --namespace dev
```