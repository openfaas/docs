# Jenkins for building OpenFaaS Functions

## Setup Jenkins on Kubernetes

Use helm to install Jenkins

```
$ helm install --name cicd stable/jenkins --set rbac.install=true
```

## Create a pipeline with Docker in Docker

Create a new Jenkins Pipeline job:

```
def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label,
containers: [
    containerTemplate(privileged: true, name: 'dind', image: 'docker:dind', ttyEnabled: true, command: 'cat')
  ]) {
    node(label) {
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/alexellis/openfaas-cloud-test']]])

        stage("Docker info")  {
            container("dind") {
                sh "apk add --no-cache curl git && curl -sLS cli.openfaas.com | sh"
                sh "dockerd &"
                sh "docker info"
                sh "faas-cli template store pull node8-express"
                sh "faas-cli build"
                sh "killall dockerd"
            }
        }
    }
}
```

You should replace `https://github.com/alexellis/openfaas-cloud-test` with your own Git repository.

If you have custom templates then run `faas-cli template store pull <template>` before `faas-cli` build.

> Note: the following example should not be used in a production cluster due to the use of a privileged container to build the Docker image. A tool such as [Kaniko from Google](https://github.com/GoogleContainerTools/kaniko) could be used do perform a non-privileged build, but is still not suitable for building untrusted code.
