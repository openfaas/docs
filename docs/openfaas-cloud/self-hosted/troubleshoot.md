## Troubleshooting OpenFaaS Cloud

### Got something wrong?

At any time you can reset the cluster and start over. This is better than editing files individually and prevents inadvertently missing something.

#### Not found error for dashboard

You cannot access the dashboard or gateway via IP address. You need to use the DNS A records for your cluster which correspond to the root domain used for your installation.

Find the IP address of the LoadBalancer or IP address of one of your nodes if using Host networking and create these records:

* `system.example.com` -> Create DNS A record `system.example.com`
* `auth.system.example.com` -> Create DNS A record `system.example.com`
* `user.example.com/function` -> Create as DNS A record `*.example.com`

If you do not have access to DNS you can edit your `/etc/hosts` file and add entries there for local testing.

#### Issues with TLS

TLS is provided by a wildcard certificate with [cert-manager](https://cert-manager.readthedocs.io/en/latest/), you can read the [documentation for cert-manager](https://cert-manager.readthedocs.io/en/latest/) for troubleshooting.

Useful commands:

* View Certificates and status

  ```sh
  kubectl get certificate -n openfaas

  kubectl describe -n openfaas certificate/name
  ```

* View the ClusterIssuer and its status

  ```sh
  kubectl get ClusterIssuer

  kubectl describe ClusterIssuer
  ```


* View any Orders that may be in-progress

  ```sh
  kubectl get Order -n openfaas

  kubectl describe -n openfaas order/name
  ```

* View Ingress objects

  ```sh
  kubectl get ingress -n openfaas
  kubectl describe -n openfaas ingress/name
  ```

#### Something else not working?

Make sure that you clearly followed all the instructions in the [README for ofc-bootstrap](https://github.com/openfaas-incubator/ofc-bootstrap/blob/master/README.md), read them through and check them off one-by-one.

Make sure that you edited `init.yaml`, read all the comments and updated the values that you needed to. If in doubt, ask questions on Slack.

### No functions appear

If no functions appear on your dashboard then try the following.

#### No GitHub status checks

Start by ensuring you have installed your GitHub App on your chosen GitHub repo and that you picked the correct permissions for your GitHub App.

#### stack.yml not found

You must name your function's YAML file `stack.yml`, you can place multiple function definitions within the same file.

#### Wrong branch

By default only the `master` branch is built, unless this was configured to point at a separate branch during the installation of OFC.

#### Custom templates

If you have a custom template, you can redeploy the `git-tar` function or edit its Kubernetes deployment.

```
kubectl edit -n openfaas-fn deploy/git-tar
```

Look at the `custom_templates` environment variable, in the `value:` field append the repo URL for your template.

You can specify multiple git repositories, just use commas to separate each. Do not use any spaces.

```yaml
    spec:
      containers:
      - env:
        - name: custom_templates
          value: https://github.com/openfaas-incubator/node10-express-template.git,https://github.com/openfaas-incubator/ruby-http
```

#### SealedSecret name mis-match

See the [secrets](https://github.com/openfaas/openfaas-cloud/tree/master/docs#sealedsecret-support) reference on how to configure your SealedSecrets correctly.

#### Webhook secret / HMAC incorrect

The webhook secret used by GitHub and your cluster must match.

#### GitHub App Advanced page

For anything else head over to your GitHub App and check the "Advanced" page.

Look for a `push` event and examine the request/response. If you need to you can redeliver a failed message.

##### TLS errors

Your TLS certificates may be inappropriately configured. Check the logs of cert-manager to see if it was able create the certificate for your domain.

##### Timeout

You may need to adjust the maximum timeout for building, pushing and deploying a function.

##### GitHub App private key or AppID mismatch

Cross-check your GitHub App's private key and AppID with the one loaded into the cluster or in `init.yaml`.

### Container logs

Check the following logs

#### system-github-event

Did the event from GitHub get handled correctly? Is there a mismatch on the webhook secret / HMAC SHA?

```sh
kubectl logs -n openfaas-fn deploy/system-github-event
```

#### buildkit

Is BuildKit functioning correctly and pushing to the remote registry?

```sh
kubectl logs -n openfaas deploy/of-builder -c of-builder
kubectl logs -n openfaas deploy/of-builder -c of-buildkit
```

If your credentials or registry are set incorrectly, you may see that of-builder passes successfully, but of-buildkit may show an authorization error.

#### Updating or fixing the registry secrets

You may not have followed the instructions that say: "do not store your Docker password in a keychain", check this by fetching the registry secret and inspecting it, if it's correct you'll see your username and password encoded in the resulting JSON file.

```sh
kubectl get secret -n openfaas registry-secret -o jsonpath='{.data.config\.json}'|base64 --decode
```

You are looking for something like this:

```json
{
        "auths": {
                "https://index.docker.io/v1/": {
                        "auth": "dXNlcjpwYXNzd29yZAo=="
                }
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/19.03.2 (darwin)"
        }
}
```

Update your Docker keychain settings then regenerate your `~/.docker.config` file. Remove the following setting if you see it in the file: `	"credsStore": "osxkeychain"`.

Once complete, run the following and edit the `config.json` section of the following two secrets:

* `kubectl get secret -n openfaas registry-secret`
* `kubectl get secret -n openfaas-fn registry-pull-secret`

Replace the text inside:

```json
    "data": {
        "config.json": ""
```

With the result from: `cat ~/.docker/config.json | base64`

#### git-tar

git-tar takes the user's code and creates a tarball to be built, are there any errors?

```sh
kubectl logs -n openfaas-fn deploy/git-tar
```

#### buildshiprun

Are there any issues deploying? Look for non 2xx status codes.

```sh
kubectl logs -n openfaas-fn deploy/buildshiprun
```

#### github-status

Were there issues updating the GitHub statuses?

```sh
kubectl logs -n openfaas-fn deploy/github-status
```

If so, then perhaps your GitHub App doesn't have the correct permissions.

#### OAuth / Auth

Check the auth service:

```sh
kubectl describe -n openfaas deploy/auth
kubectl logs -n openfaas deploy/auth
```

Check that `client_id` is set correctly along with the direct URL and cookie domain.

### Still not working?

Head over to [OpenFaaS Slack](/community/) and join #openfaas-cloud.