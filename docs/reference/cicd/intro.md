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

* The `--shrinkwrap` flag

    The `faas-cli build` command invokes the `docker` CLI with the various flags and parameters required. If you want to use an alternative builder you can use the `--shrinkwrap` flag to generate a folder named `./build/<function>` which can then be used with any other container builder such as [BuildKit](https://github.com/moby/buildkit) or [Kaniko](https://blog.alexellis.io/quick-look-at-google-kaniko/).

See also: [`faas-cli build` reference](../../cli/build/).

### GitLab

GitLab is both a source-control management system and a CI tool.

You can find a pipeline example on the [GitLab CI/CD page](../gitlab/).

### GitHub

For GitHub you can build with any suitable CI tool such as:

* [Jenkins](https://jenkins.io)
* [Travis CI](https://travis-ci.org)
* [Codefresh](https://codefresh.io)
* [Drone CI](https://drone.io)
* [Circle CI](https://circleci.com/)
* [JenkinsX](https://jenkins.io/projects/jenkins-x/)
* [GitHub Actions](https://github.com/features/actions) (beta)
* [Tekton pipelines](https://github.com/tektoncd/pipeline) (experimental)

It's really up to you.

### Git examples with Jenkins Pipeline

Examples with Docker-in-Docker: [Jenkins Pipeline examples](../jenkins/)

### OpenFaaS Cloud

OpenFaaS Cloud can integrate with a public GitHub account or organisation or self-hosted GitLab instance to provide automatic CI/CD without any additional maintenance.

See also: [OpenFaaS Cloud](../../../openfaas-cloud/intro/)
