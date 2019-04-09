## Configure self-hosted GitLab for OpenFaaS Cloud

This guide is for connecting your own self-hosted GitLab instance to your OpenFaaS Cloud deployment.

Use this guide to configure your `init.yaml` file for use with [ofc-bootstrap](https://github.com/openfaas-incubator/ofc-bootstrap).

### Edit your init.yaml file

* Edit `init.yaml` and set `gitlab_instance:` to the public address of your GitLab instance.

* Edit `init.yaml` and set `scm: github` to `scm: gitlab`.

#### Create an access token

You will need to create an access token with `sudo` permissions and API access.

* Go to your Personal profile
* Click *Access Tokens*
* Create a token
* Name: OpenFaaS Cloud
* Scopes: `api, read_repository, sudo, read_user`

You will be given an API token at this time and must enter it into init.yaml.

Look for: `"gitlab-api-token"` and set the `value` to the value from the UI

#### Create the system hook

This hook will publish events when there is a `git push` into a repo.

* Go to your Admin area.
* Click *System Hooks*
* Enter the URL `https://system.domain.com/gitlab-event` replacing `domain.com` with your domain
* Do not enter a value for *Secret Token* at this time

![System hooks](/images/ofc-gitlab/system-hook.png)

#### Create an OAuth application

The OAuth application will be used for logging in to your dashboard.

* Click Admin Area then Applications
* Click New application
* Under Name enter "OpenFaaS Cloud for GitLab"
* In Redirect URI enter "https://auth.system.domain.com/oauth2/authorized"

* Add the scopes as pictured below

![System hooks](/images/ofc-gitlab/scopes.png)

After creating your application you will get a client_id and client_secret.

Set your (Application Id) client_id in `init.yaml` under: `oauth` and `client_id`

Set your (Secret) client_secret in `init.yaml` under: `"of-client-secret"`.

Set `oauth_provider_base_url` to `https://gitlab.domain.com`, where this is the domain of your GitLab instance. Don't add a final slash to the URL.

Set `enable_oauth` to `true`
