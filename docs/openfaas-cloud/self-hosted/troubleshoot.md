## Troubleshooting OpenFaaS Cloud

### No functions appear

If no functions appear on your dashboard then try the following.

#### No GitHub status checks

Start by ensuring you have installed your GitHub App on your chosen GitHub repo and that you picked the correct permissions for your GitHub App.

#### stack.yml not found

You must name your function's YAML file `stack.yml`, you can place multiple function definitions within the same file.

#### Wrong branch

You must use the `master` branch. You cannot deploy from any other branch at this time.

#### SealedSecret name mis-match

See the [secrets](./secrets) reference on how to configure your SealedSecrets correctly.

#### Webhook secret / HMAC incorrect

The webhook secret used by GitHub and your cluster must match.

#### GitHub App Advanced page

For anything else head over to your GitHub App and check the "Advanced" page.

Look for a `push` event and examine the request/response. If you need to you can redeliver a failed message.

##### SSL error

Your SSL certificates may be inappropriately configured. Check cert-manager.

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

#### buildshiprun

Are there any issues deploying?

```sh
kubectl logs -n openfaas-fn deploy/buildshiprun
```

#### github-status

Were there issues updating the GitHub statuses?

```sh
kubectl logs -n openfaas-fn deploy/github-status
```
