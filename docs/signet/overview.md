# Signet - OpenID Connect Provider

Signet is a lightweight, self-contained OpenID Connect (OIDC) provider for
automated demos, integration tests, single-node Kubernetes environments, and
small production deployments.

Deploy Signet for:

- [SSO / IAM with OpenFaaS for Enterprises](/openfaas-pro/iam/overview/)
- [OAuth login for OpenFaaS functions](/reference/function-oauth/)
- An identity provider for [Inlets Pro tunnel authentication](https://docs.inlets.dev/tutorial/http-authentication/)
- [OAuth for the Inlets Uplink API](https://docs.inlets.dev/uplink/rest-api/)
- A standalone OpenID Connect (OIDC) provider for any application

This walkthrough covers the installation of Signet on Kubernetes with the
Helm chart, published as an OCI artifact: `oci://ghcr.io/openfaasltd/signet-provider`

## Deploy Signet

### Prerequisites

* Requires a license for one of the following: [OpenFaaS Pro/For Enterprises](https://www.openfaas.com/pricing/), [Superterm](https://superterm.dev), [Slicer](https://slicervm.com), or [Inlets Pro](https://inlets.dev).
* A Kubernetes cluster and Helm 3.
* [arkade](https://github.com/alexellis/arkade) - a tool for downloading and installing developer binaries

    To install arkade run:
    ```
    curl -sSLf https://get.arkade.dev/ | sudo sh
    ```

### 1) Create a values file

Set `issuer` to the public OIDC URL that Signet should use in tokens and its
discovery document. Signet reads it on first run, so changing it later requires
a store reset. See [Exposing the issuer](#exposing-the-issuer) for the options.

Create a `values.yaml` file:

```yaml
issuer: "https://signet.example.com"
```
Adding initial users and clients is optional. To seed them before the first install, use the
`config` parameter described in [Provisioning identities](#option-c-seed-at-install-time).

View the full default values.yaml file:

```bash
helm show values oci://ghcr.io/openfaasltd/signet-provider
```

Preview the rendered Kubernetes manifests with your values:

```bash
helm template signet oci://ghcr.io/openfaasltd/signet-provider \
  --namespace signet \
  -f ./values.yaml
```

Add the `--version` flag to get a specific chart version. See the [package page](https://github.com/orgs/openfaasltd/packages/container/package/signet-provider)
for available chart versions.

All the various configuration options for the Helm chart are documented in the [Configuration](#configuration) section.

### 2) Create the licence Secret

The chart mounts the licence from a Secret named `signet-license` with the key
`license`. Create it before installing:

```bash
kubectl create namespace signet
kubectl create secret generic \
  -n signet \
  signet-license \
  --from-file license=$HOME/.openfaas/LICENSE
```

### 3) Install the chart

```sh
helm upgrade signet \
  --install oci://ghcr.io/openfaasltd/signet-provider \
  --namespace signet \
  -f ./values.yaml
```

If you want to pin the version of the Helm chart, you can do so with the `--version` flag.

By default, the chart generates a random 32-byte master key and admin token on
install, and stores them in Secrets named `signet-master` and `signet-admin-token` with
`helm.sh/resource-policy: keep`, so they survive upgrades and uninstall.

### 4) Retrieve the admin token

The admin (management) token is retrievable at any time:

```sh
kubectl -n signet get secret signet-admin-token \
  -o jsonpath='{.data.admin-token}' | base64 -d
```

### Verify the installation

```sh
kubectl -n signet rollout status deployment/signet
kubectl -n signet get svc signet
```

With the default ClusterIP Service, use port-forwarding to try it out:

```sh
kubectl -n signet port-forward svc/signet 8080:8080 &
export SIGNET_URL=http://127.0.0.1:8080
curl $SIGNET_URL/healthz
open $SIGNET_URL/.well-known/openid-configuration
```

If you set `issuer` to an external URL, use that instead of the port-forward:

```sh
export SIGNET_URL=https://signet.example.com
```

## Provisioning identities

Signet starts with an empty store. Add users and OAuth clients by seeding them
at install time, or manage them afterward with the Signet CLI or admin API.

### Option A: Signet CLI

Install the Signet CLI on your workstation with arkade:

```sh
arkade get signet
signet version
```

The CLI connects to the running Signet deployment through its admin API.
Set `SIGNET_URL` to the issuer URL where Signet is available, and retrieve its
admin token:

```sh
export SIGNET_URL=https://signet.example.com
export SIGNET_ADMIN_TOKEN=$(kubectl -n signet get secret signet-admin-token \
  -o jsonpath='{.data.admin-token}' | base64 -d)
```

Create a user and an OAuth client:

```sh
signet user add --groups admin admin
signet client add --redirect-url https://dashboard.openfaas.example.com/auth/callback openfaas
```

Replace the redirect URL with the callback URL of the application that will
authenticate against Signet. Signet generates and prints the user's password and
client secret. Save them somewhere safe.

List users and clients:

```sh
signet user list
signet client list
```

Use a public client for browser, mobile, or desktop apps that cannot keep a
client secret private. These apps must support Proof Key for Code Exchange
(PKCE):

```sh
signet client add --public --redirect-url https://app.example.com/callback web
```

Use `signet user rm <username>` or `signet client rm <id>` to remove an
identity. Run `signet user add --help` or `signet client add --help` for the
available flags.

### Option B: admin API

Use the admin token from [step 4](#4-retrieve-the-admin-token) as a Bearer
token against `/admin/users` and `/admin/clients`:

```sh
curl -s \
  -H "Authorization: Bearer $SIGNET_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"change-me","name":"Signet Admin","email":"admin@example.com","groups":["admin"]}' \
  $SIGNET_URL/admin/users

curl -s \
  -H "Authorization: Bearer $SIGNET_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"id":"openfaas","secret":"change-me","redirect_urls":["https://dashboard.openfaas.example.com/auth/callback"]}' \
  $SIGNET_URL/admin/clients
```

Both endpoints also support `GET` (list), `PATCH` (update) and `DELETE`
(remove) per user/client.


### Option C: seed at install time

Add the users and clients directly to `values.yaml` before the first
install:

```yaml
config: |
  {
    "users": [
      {
        "username": "admin",
        "password": "change-me",
        "subject": "admin",
        "name": "Signet Admin",
        "email": "admin@example.com",
        "groups": ["admin"]
      }
    ],
    "clients": [
      {
        "id": "openfaas",
        "secret": "change-me",
        "redirect_urls": ["https://dashboard.openfaas.example.com/auth/callback"]
      }
    ]
  }
```

> Note: the seed config is imported on the **first run only**, when the state
> store is empty. A later `helm upgrade` with a changed config will not
> re-provision identities. Use the CLI or admin API for ongoing management, or reset
> the store (see [Removing Signet](#removing-signet)).

## Exposing the issuer

Signet is both an IdP and a web UI, so the issuer URL must be reachable from browsers. The in-cluster Service DNS name like `http://signet.signet.svc:8080` only works for in-cluster callers such as OpenFaaS, not for browsers.

* For CI, agents, and e2e tests, port-forwarding is the quickest option:
  `kubectl -n signet port-forward svc/signet 8080:8080` and use
  `http://127.0.0.1:8080` as the issuer.
* For bare-metal or single-node clusters, a NodePort can be used:
  `service.type=NodePort` and `service.nodePort=30080`, then use
  `http://<node-ip>:30080` as the issuer.
* For production, put an Ingress or load balancer with TLS in front of Signet
  and use that HTTPS URL as the issuer.

## Federated login with GitHub (optional)

Signet can delegate identity to GitHub via OAuth Device Flow. A `client_id`
only, no client secret or redirect URL to register.

Authorization is a pure allowlist and default deny, so provide at least one of `allowedLogins` or
`allowedOrgs` or nobody can sign in.

```yaml
github:
  clientId: Ov23li...
  # GitHub login names (usernames), not email addresses
  allowedLogins:
    - alexellis
```

Like the users and clients seed, the `github` block in `values.yaml` is
imported on the first run only. Later `helm upgrade`s will not re-apply
it, and values never override policy set through the API.

To enable or edit federation at any time, use the admin API, which updates
the policy live: no restart or store reset required.

```sh
export SIGNET_ADMIN_TOKEN=$(kubectl -n signet get secret signet-admin-token \
  -o jsonpath='{.data.admin-token}' | base64 -d)

curl -s \
  -H "Authorization: Bearer $SIGNET_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -X PATCH \
  -d '{"client_id":"Ov23li...","allowed_logins":["alexellis"]}' \
  $SIGNET_URL/admin/github
```

`PATCH` updates `client_id`, `allowed_logins` and `allowed_orgs`, and fields
left out keep their current value. `GET` returns the current policy and
`DELETE` disables GitHub sign-in.

Users then pick *Sign in with GitHub* on the login page and enter the device
code shown at https://github.com/login/device.

## Removing Signet

```sh
helm uninstall signet -n signet
kubectl -n signet delete secret signet-master signet-admin-token signet-license --ignore-not-found
kubectl -n signet delete pvc signet-state --ignore-not-found
```

Helm retains both chart-generated credential Secrets on uninstall, but removes the PVC.
Check your storage provisioner's reclaim policy if PVC data must survive.
The commands above also delete the default Secrets. If you configured custom
Secret names, delete those separately when they are no longer needed.

Delete both the PVC and master key Secret to reset. Reinstalling then creates
a new store from the configured seed.

## Configuration

Specify each parameter using the `--set key=value[,key=value]` argument to
`helm install`, or add them to `values.yaml`. To inspect the chart's
default values:

```sh
helm show values oci://ghcr.io/openfaasltd/signet-provider
```

### General parameters

| Parameter | Description | Default |
| ----------------------- | ---------------------------------------------------------- | ---------------------------------------------------------- |
| `image` | Full container image including tag, a single string | See Helm values |
| `registryPrefix` | Adds a prefix or replaces the registry prefix of the image | `""` |
| `imagePullPolicy` | Image pull policy | `IfNotPresent` |
| `replicaCount` | Replicas of Signet, more than one requires a shared volume | `1` |
| `issuer` | OIDC issuer URL embedded in tokens and the discovery document, read on the first run. The generated default is only reachable inside the cluster | `""` (generates `http://<service-name>.<namespace>.svc:<service-port>`) |
| `listen` | Address the Signet server listens on | `:8080` |
| `config` | Optional seed config JSON imported on the store's first run: users and clients | `""` |
| `github.clientId` | GitHub OAuth App client id for device flow federation | `""` |
| `github.allowedLogins` | Allowlist of GitHub usernames, default deny | `[]` |
| `github.allowedOrgs` | Allowlist of GitHub organizations, default deny | `[]` |
| `masterKey.secretName` | Pre-created Secret containing `master.key` (exactly 32 bytes). When unset, the chart generates and retains a Secret | `""` |
| `adminToken.secretName` | Pre-created Secret containing a non-empty `admin-token`. When unset, the chart generates and retains a Secret | `""` |
| `license.secretName` | Name of the pre-created Secret containing the `license` key | `signet-license` |
| `service.type` | Type of Service to create `ClusterIP/NodePort/LoadBalancer` | `ClusterIP` |
| `service.port` | Port exposed by the Service | `8080` |
| `service.nodePort` | Optional NodePort, when `service.type` is `NodePort` | `""` |
| `storage.className` | StorageClass for the state PVC | `local-path` |
| `storage.size` | Size of the state PVC | `1Gi` |
| `storage.mountPath` | Where the state PVC is mounted, passed as `--state-dir` | `/var/lib/signet` |
| `securityContext.runAsUser` | UID to run the container as | `65532` |
| `securityContext.runAsNonRoot` | Enforce non-root user | `true` |
| `resources` | Resource requests and limits for the Signet container | `{}` |
