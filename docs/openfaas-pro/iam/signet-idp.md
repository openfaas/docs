# OpenFaaS IAM with Signet IdP

This walkthrough uses [Signet](/signet/overview/) as the identity provider for
OpenFaaS IAM.

We will create a user and an OAuth client in Signet, then use subject, group,
and email claims to grant access to functions and secrets through a Role and Policy.

## Prerequisites

* OpenFaaS for Enterprises with [IAM enabled](/openfaas-pro/iam/overview/#installation).
* A [Signet installation](/signet/overview/) with an HTTPS URL reachable from your browser and the OpenFaaS components.
* The [Signet CLI](/signet/overview/#option-a-signet-cli), [faas-cli](/cli/install/), and [Pro plugin](/cli/install/#pro-plugin).
* Administrator access through `kubectl` to create the IAM resources.

Replace `https://signet.example.com` and `https://gateway.openfaas.example.com`
throughout the examples with your Signet and OpenFaaS gateway URLs.

## Create a user and a client in Signet

Connect the Signet CLI to your installation and retrieve the admin token
from the default secret in the `signet` namespace:

```sh
export SIGNET_URL=https://signet.example.com
export OPENFAAS_URL=https://gateway.openfaas.example.com
export SIGNET_ADMIN_TOKEN=$(kubectl -n signet get secret signet-admin-token \
  -o jsonpath='{.data.admin-token}' | base64 -d)
```

Create a new user named `alice` and assign her to the `openfaas-dev` group:

```sh
signet user add \
  --name "Alice Smith" \
  --email alice@example.com \
  --groups openfaas-dev \
  alice
```

The command prints a generated password. Save it to test logging in later.

Create a client for faas-cli:

```sh
signet client add --public \
  --redirect-url http://127.0.0.1:31111/oauth/callback \
  openfaas
```

## Register the Issuer

Add your Signet instance as a trusted issuer to OpenFaaS:

```yaml
apiVersion: iam.openfaas.com/v1
kind: JwtIssuer
metadata:
  name: signet
  namespace: openfaas
spec:
  iss: https://signet.example.com
  aud:
    - openfaas
  tokenExpiry: 12h
```

Use your Signet URL for `iss` and the client ID for `aud`.

See the [Signet SSO setup](/openfaas-pro/sso/signet/) for full details on client
configuration and issuer registration.

## Create a Policy and Role

The Policy allows management of functions and secrets in the existing `dev`
namespace. All IAM resources should be created in the `openfaas` namespace.

Save the following as `signet-policy.yaml`:

```yaml
apiVersion: iam.openfaas.com/v1
kind: Policy
metadata:
  name: signet-dev-rw
  namespace: openfaas
spec:
  statement:
    - sid: manage-dev
      action:
        - Function:List
        - Function:Get
        - Function:Create
        - Function:Update
        - Function:Delete
        - Function:Logs
        - Function:Scale
        - Secret:List
        - Secret:Create
        - Secret:Update
        - Secret:Delete
      effect: Allow
      resource:
        - "dev:*"
```

```sh
kubectl apply -f signet-policy.yaml
```

See [IAM permissions](/openfaas-pro/iam/overview/#permissions) for an overview
of all supported actions and resource scopes.

### Match on subject

We will bind the Policy to a Role that matches Alice's `sub` claim. Signet sets
this claim to the username, `alice`, from the user creation command.

Save the following as `signet-role.yaml`:

```yaml
apiVersion: iam.openfaas.com/v1
kind: Role
metadata:
  name: signet-developers
  namespace: openfaas
spec:
  policy:
    - signet-dev-rw
  principal:
    jwt:sub:
      - alice
  condition:
    StringEqual:
      jwt:iss: ["https://signet.example.com"]
```

The `principal` matches Alice's subject, and the condition checks that the token
comes from your Signet instance. Alice matches the Role when both checks pass.

```sh
kubectl apply -f signet-role.yaml
```

### Match on email

We can also match the `email` claim, which contains the address set with `--email`
when creating the user. For Alice, this is `alice@example.com`.

Update `signet-role.yaml` to match the email address:

```yaml
apiVersion: iam.openfaas.com/v1
kind: Role
metadata:
  name: signet-developers
  namespace: openfaas
spec:
  policy:
    - signet-dev-rw
  condition:
    StringEqual:
      jwt:iss: ["https://signet.example.com"]
      jwt:email: ["alice@example.com"]
```

Both the issuer and email address must match for the user to match the Role.

```sh
kubectl apply -f signet-role.yaml
```

### Match on group membership

Instead of matching on `sub`, we can match the `groups` claim to grant the same
permissions to members of `openfaas-dev`.

Signet includes the groups assigned with `--groups` in the user's token. For Alice,
the `groups` claim contains `openfaas-dev`.

Update `signet-role.yaml` to match the `groups` claim instead of `sub`:

```yaml
apiVersion: iam.openfaas.com/v1
kind: Role
metadata:
  name: signet-developers
  namespace: openfaas
spec:
  policy:
    - signet-dev-rw
  condition:
    StringEqual:
      jwt:iss: ["https://signet.example.com"]
    ForAnyValue:StringEqual:
      jwt:groups: ["openfaas-dev"]
```

`ForAnyValue:StringEqual` matches `openfaas-dev` in the token's `groups` array.
The issuer must also match. Any user in this group matches the Role.

```sh
kubectl apply -f signet-role.yaml
```

## Authenticate and verify access

We will now sign in as Alice and verify that she can deploy and manage functions
in the `dev` namespace.

Log in to OpenFaaS:

```sh
faas-cli pro auth \
  --gateway "$OPENFAAS_URL" \
  --authority "$SIGNET_URL" \
  --client-id openfaas
```

In the browser, sign in as `alice` using the password printed when you created
the user.

The CLI exchanges the Signet token for an OpenFaaS access token and
saves it for subsequent commands.

Alice can now deploy and list functions in `dev`:

```sh
faas-cli store deploy nodeinfo --namespace dev
faas-cli list --namespace dev
```

## Inspect the Signet claims

To see all the claims available in a Signet token, use the CLI. The `--no-exchange`
flag skips exchanging it for an OpenFaaS token, and `--pretty` shows its decoded
contents:

```sh
faas-cli pro auth \
  --gateway "$OPENFAAS_URL" \
  --authority "$SIGNET_URL" \
  --client-id openfaas \
  --print-token \
  --pretty \
  --no-exchange
```

Example decoded token for Alice:

```json
{
  "header": {
    "alg": "ES256",
    "kid": "signet-1",
    "typ": "JWT"
  },
  "payload": {
    "aud": [
      "openfaas"
    ],
    "auth_time": 1791385368,
    "email": "alice@example.com",
    "exp": 1791388968,
    "groups": [
      "openfaas-dev"
    ],
    "iat": 1791385368,
    "idp": "signet",
    "iss": "https://signet.example.com",
    "name": "Alice Smith",
    "nonce": "1791385354124866000",
    "sub": "alice"
  }
}
```
