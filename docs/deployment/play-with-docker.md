# Deployment guide for the Docker Playground (PWD)

> Note: You will need a free [Docker Hub](https://hub.docker.com) account to access play-with-docker.

## 1.0 Start a new session

Go to [play-with-docker.com](https://labs.play-with-docker.com) and log in with your Docker Hub account

This will create a new 4 hour session with access to all of docker's features in your browser

Once logged in, click the "+ ADD NEW INSTANCE" link on the right-hand side

This will open a console with Docker ready to run

## 1.0 Initialize Swarm Mode

In the console, run the swarm `init` command to initialize Swarm mode

```bash
docker swarm init --advertise-addr eth0
```

Highlight the join token command that is output and right-click - "copy" (we'll need that to join additional worker nodes)

## 1.1 Join workers (optional)

> OpenFaas works well for experimentation with only the master node if you don't want to create workers

Click the "+ ADD NEW INSTANCE" link on the right-hand side again to create another docker host within your session

In the new console, right-click and paste the join token command that was just copied

Repeat these steps for as many workers as you would like.

## 2.0 Deploy the stack

Go back to the master node by clicking the first button on the right-side menu

> This should have an icon of a person/user to indicate it is a swarm master

In the console, clone OpenFaaS and deploy the stack by copy/pasting the following command:

```bash
git clone https://github.com/openfaas/faas && \
  cd faas && \
  ./deploy_stack.sh
```

`./deploy_stack.sh` can be run at any time and deploys the core OpenFaas components. You can read more about these in the [TestDrive document](https://github.com/openfaas/faas/blob/master/TestDrive.md)

## 2.1 Test out the UI

Within a few seconds (or minutes depending on PWD's load at the time) the API gateway will be deployed and you will be able to access the UI

Once the command is complete, you will notice two links appear at the top of the page with port numbers

`8080` and `9090`

To view the OpenFaaS UI, simply click the `8080` link.

This will open a new tab in your browser with the UI from your PWD Swarm cluster

The `9090` link will open the Prometheus console UI. More on that in the [Auto-scaling](/architecture/autoscaling/) section

## 3.0 Start the hands-on labs

Learn how to build serverless functions with OpenFaaS and Python in our half-day workshop. You can follow along online at your own pace.

* [OpenFaaS workshop](/tutorials/workshop/)

> There are **only 4 hours** available per PWD session, but we've found that most people are able to walk through the workshop in less time.

## Troubleshooting

If you are running into any issues please check out the troubleshooting guide and search the documentation / past issues before raising na issue.

* [Troubleshooting guide](https://github.com/openfaas/faas/blob/master/guide/troubleshooting.md)

Alternatively, just click the orange "CLOSE SESSION" button under the timer and start over with a fresh session
