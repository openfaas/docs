## Multi-stage OpenFaaS Cloud

If you would like to create a buffer between pushing code to master and deploying new endpoints to production, then you may benefit from multiple stages or environments for development, staging, and production.

### Reference architecture

The following architecture can be used for dual-stage configuration for staging and production.

The code for the purchase service is deployed from a single repository, but the secrets for either environment are stored in separate repositories.

![](/images/openfaas-cloud/multi-stage.png)

Production is deployed from the `production` branch.

![](/images/openfaas-cloud/multi-stage-2.png)

Staging is deployed from the `staging` branch.

#### Configuration

Install OpenFaaS Cloud with [ofc-bootstrap](https://github.com/openfaas/ofc-bootstrap), once for each environment on separate clusters. Some configuration in `init.yaml` needs to be changed, other settings can remain the same:

* Set the `root_domain`

Set a separate sub-domain for each environment to isolate your endpoints

* Set the `build_branch`

The `build_branch` configures the pipeline to respond to build events from that particular branch of connected repositories.

* GitHub / GitLab configuration

You will need a different GitHub App and GitHub OAuth App with its own private key and URL.

You can keep everything else the same including:

* Docker registry prefix

The URLs used for Docker images contains the build branch, so you can use the same registry and won't encounter conflicts.

#### Secrets

Each OpenFaaS Cloud cluster has its own private and public key-pair for SealedSecrets. Your secret values may be different for production and staging. You can manage them in the following way:

* Single stage environments (i.e. Community Cluster)

If you only have one environment such as in the Community Cluster, then secrets can be stored in the same repository as your application code. As soon as you have more than one environment with different secrets or configuration, then it becomes hard to determine which secret to use.

* Keep secrets in one or more git repositories for each stage (recommended)

It is recommended that you create an attach one or more git repositories with your `secrets.yaml` file per environment.

Here is an example for a single code repository and two secrets repositories:

| Environment      | Repo                         | Contents          |
|------------------|------------------------------|-------------------|
| Staging          | acmeco/staging-secrets       | `secrets.yaml`    |
| Production       | acmeco/prod-secrets          | `secrets.yaml`    |
| Both             | acmeco/payment-service       | `stack.yaml`, `payment/handler.js`, `payment/package.json`      |

To generate the secrets you would download both public keys and run the following:

Within the `acmeco/staging-secrets` repo run:

```sh
faas-cli cloud seal --name secrets \
  --literal mongo-username="alexellis" \
  --literal mongo-password="test" \
  --literal mongo-uri="mongodb://mongodb-us-east-1.aws.com" \
  --cert staging.pem
```

Then commit your `secrets.yaml` file.

Within the `acmeco/prod-secrets` repo run:

```sh
faas-cli cloud seal --name secrets \
  --literal mongo-username="alexellis" \
  --literal mongo-password="test" \
  --literal mongo-uri="mongodb://mongodb-us-east-1.aws.com" \
  --cert prod.pem
```

Then commit your `secrets.yaml` file.

Within the `acmeco/payment-service` repo:

In `stack.yml`:

```yaml
functions:
  payment:
    handler: ./payment
    image: payment:0.1.0
    secrets:
      - secrets
```

Within `payment/handler.js`:

```js
const mongoURI = await fs.readFile("/var/openfaas/secrets/mongo-uri")
const mongoUsername = await fs.readFile("/var/openfaas/secrets/mongo-username")
const mongoPassword = await fs.readFile("/var/openfaas/secrets/mongo-password")
```

In both environments, the secret names will be the same, but the values can vary as necessary.

Then install the GitHub App or GitLab integration for both secrets repositories first, then the code repository next.
