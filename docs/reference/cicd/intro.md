## CI/CD with OpenFaaS

Due to the fact that OpenFaaS functions are built into portable Docker images you can use any container builder to build your functions. The `faas-cli` can be used to `build`, `push` and `deploy` your functions.

### Use the native `faas-cli`

It is recommended to use the `faas-cli` binary for building and deploying your functions whether that is to Kubernetes or faasd.

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

See also: [`faas-cli build` reference](/cli/build/).

### Building via REST API

Customers of OpenFaaS Pro can make use of our in-cluster builder.

Send your code as generated with `faas-cli build --shrinkwrap` to the `/build` endpoint of the OpenFaaS Pro builder and receive a JSON response with build logs and a URL to the published image.

Learn more: [Pro Builder API](/openfaas-pro/builder/)

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

### GitHub Actions

* [Build and deploy OpenFaaS functions with GitHub Actions](https://www.openfaas.com/blog/openfaas-functions-with-github-actions/) By Utsav Anand
