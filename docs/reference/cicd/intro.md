## CI/CD with OpenFaaS

Due to the fact that OpenFaaS functions are built into portable Docker images you can use any container builder to build your functions. The `faas-cli` can be used to `build`, `push` and `deploy` your functions.

### Use the native `faas-cli`

It is recommended to use the `faas-cli` binary for building and deploying your functions whether that is to Kubernetes or Docker Swarm.

* Build only:
    ```
    faas-cli build
    ```

* Build only & push:

    ```
    faas-cli build
    faas-cli push
    ```

* To combine build, push and deploy:
    ```
    faas-cli up
    ```

You can also use `--parallel` or / `--filter` when you have multiple functions in your `stack.yml` file.

See the reference for the `faas-cli build` command [here](../../cli/build/).

### GitLab

Get a [sample configuration for GitLab](./gitlab/).

### GitHub

For GitHub you can build with any suitable CI tool such as [Jenkins](https://jenkins.io), [JenkinsX](https://jenkins.io/projects/jenkins-x/), [Travis CI](https://travis-ci.org), [Drone CI](https://drone.io), [Tekton pipelines](https://github.com/tektoncd/pipeline), [Circle CI](https://circleci.com/) or even [GitHub Actions](https://github.com/features/actions).

It's really up to you.

### OpenFaaS Cloud

OpenFaaS Cloud can integrate with a public GitHub account or organisation or self-hoted GitLab instance to provide automatic CI/CD without any additional maintenance.

See also: [OpenFaaS Cloud](../../openfaas-cloud/intro)
