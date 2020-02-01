# SSL on Docker Swarm with Traefik

To help secure your OpenFaaS installation, you need TLS.  On [Docker Swarm](docs/reference/ssl/kubernetes-with-cert-manager.md), you can do this easily with [Traefik](https://traefik.io/) and [Let's Encrypt](https://letsencrypt.org/).  Traefik is a reverse proxy that comes with TLS support via Let's Encrypt. In this tutorial we will show you how to deploy OpenFaaS with Traefik.


### Create a DNS record

Determine the public IP address that can be used to access your cluster.
If your domain is `.example.com` then create an A record using your DNS administration panel such as `gw.example.com`.

> The required steps will vary depending on your domain provider and your cluster provider. For example; [on Google Cloud DNS](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) or [with Route53 using AWS](https://kubernetes.io/docs/setup/custom-cloud/kops/#2-5-create-a-route53-domain-for-your-cluster).

Once created, verify that what you entered into your DNS control-panel worked with `ping`:

```sh
ping gw.example.com
```

You should now see the value you entered. Sometimes DNS can take 1-5 minutes to propagate.

## Update the Compose configuration

A complete example `docker-compose.yaml` can be found [here](./compose-example.yaml)

### Configure Traefik
To use Traefik with OpenFaaS, you need to modify the OpenFaaS deployment manifest to include Traefik and configuring OpenFaaS to communicate through Traefik instead of directly exposing its services publicly.

Clone OpenFaaS and then checkout the latest stable release:

    ```bash
    $ git clone https://github.com/openfaas/faas && cd faas
    ```

Add the `traefik` service to the  `docker-compose.yaml`.

To start, open `docker-compose.yaml` in your favorite editor and add a `traefik` service like this:


```yaml
version: "3.3"
services:
    traefik:
        image: traefik:v2.1.3
        container_name: "traefik"
        command:
            - "--api.insecure=true"
            - "--providers.docker=true"
            - "--providers.docker.exposedbydefault=false"
            - "--entrypoints.web.address=:80"
            - "--entrypoints.websecure.address=:443"
            - "--certificatesresolvers.myhttpchallenge.acme.httpchallenge=true"
            - "--certificatesresolvers.myhttpchallenge.acme.httpchallenge.entrypoint=web"
            - "--certificatesresolvers.myhttpchallenge.acme.email=<your-email-here>"
            - "--certificatesresolvers.myhttpchallenge.acme.storage=/letsencrypt/acme.json"
        ports:
            - "80:80"
            - "443:443"
            - "8080:8080"
        volumes:
            - "./letsencrypt:/letsencrypt"
            - "/var/run/docker.sock:/var/run/docker.sock"
        networks:
        - functions
        placement:
            constraints: [node.role == manager]

    gateway:
        ...
```

This configuration does a few important things:

* The `--providers.docker.*` flags tell Traefik to use Docker and specify that it's running in a Docker Swarm cluster.
* The `--entryPoints` flags define entry points and protocols to be used. In our case this includes HTTP on port 80 and HTTPS on port 443.
* The `--certificatesresolvers.*` flags configure Traefik to use the ACME protocol to generate Let's Encrypt certificates to secure your OpenFaaS cluster with SSL.
Make sure to set your own email within the `certificatesresolvers.myhttpchallenge.acme.email` command line argument of the traefik service.
* Expose the ports required for Traefik.  Traefik uses port `8080` for its operations and UI, while in the last step we configured ports `80` and `443` and entrypoints for internet traffic.  These will be used / proxied to OpenFaaS
* Binds the docker socket to the traefik container so that it can communicate with the Docker API and dertermine the number of containers and their IP addressess.
* Adds `traefik` to the `functions   network so that it can communicate with the OpenFaaS components.
* Ensures that Traefik is only deployed on the Docker Swarm manager.


### Configure OpenFaaS
By default, the original `docker-compose.yaml` file exposes the OpenFaaS `gateway` on port `8080`.  This conflicts with the configuration of Traefik, additionally, we want all communication to safely pass through Traefik.  To do this you must make two modifications to the `gateway` service.

First, remove the `ports` section.

Next, add the following `labels` directive to the gateway service.

```yaml
    gateway:
        image: ...
        networks:
            ...
        environment:
            ...
        labels:
            - "traefik.enable=true"
            - "traefik.http.routers.gateway.rule=Host(`gw.example.com`)"
            - "traefik.http.routers.gateway.entrypoints=websecure"
            - "traefik.http.routers.gateway.tls.certresolver=myhttpchallenge"
        deploy:
            ...
        secrets:
            ...
    ...
```

Traefik uses reads these labels from the Docker API to detect and configure the routing for the gateway.

### Configure data volumes

Finally, while configuring Traefik, you mounted a volume called `letsencrypt`.  You must now define the `letsencrypt` volume used for storing Let's Encrypt certificates. We can define an empty volume, meaning data will not persist if you destroy the container. If you destroy the container, the certificates will be regenerated the next time you start Traefik.

Add the following volumes directive on the last line of the file:

```yaml
...
volumes:
    letsencrypt:
```

A complete example `docker-compose.yaml` can be found [here](./compose-example.yaml)

## Install OpenFaaS

With all of the modifications in place, you can now deploy using the standard deployment scripts. On Mac and Linux: `deploy_stack.sh` and on Windows use `deploy_stack.ps1`. This script will deploy all of the required resources (services, configuration files, networks, and secrets) for your OpenFaaS cluster as they are defined in the `docker-compose.yaml` file.

First, make sure that Docker will execute commands on your Swarm manager node:

```bash
eval $(docker-machine env <name-of-manager-node>)
```

Then, run your deployment script. A successful install will look like this:

```bash
$ ./deploy_stack.sh
Attempting to create credentials for gateway..
adadsf809adf098adfajlk12g
mb1jl213nlkmqfadsaokv68lh
[Credentials]
 username: admin
 password: <randomly-generated-password>
 echo -n <randomly-generated-password> | faas-cli login --username=admin --password-stdin

Enabling basic authentication for gateway..

Deploying OpenFaaS core services
Creating network func_functions
Creating config func_alertmanager_config
Creating config func_prometheus_config
Creating config func_prometheus_rules
Creating service func_alertmanager
Creating service func_traefik
Creating service func_gateway
Creating service func_faas-swarm
Creating service func_nats
Creating service func_queue-worker
Creating service func_prometheus
```

## Deploy and Invoke a function

In your projects containing OpenFaaS functions, you can now deploy using your domain as the gateway.

### Deploy from the CLI
```sh
faas-cli login --gateway https://gw.example.com --username admin --password <randomly-generated-password>
faas-cli deploy --gateway https://gw.example.com
```
Replace `gw.example.com` with your domain as well as adding the username `admin` and secure random password that the deploy script created for you when you deployed OpenFaaS.

### Use the web UI
You can use the web UI to see the functions deployed to your cluster or to deploy functions from the Store.  In your web browser, go to https://gw.example.com/ui/. Note that the trailing slash is required.

On your first visit, the HTTP authentication dialogue box will open, you can login with the username `admin` and secure random password that the deploy script created for you when you deployed OpenFaaS.

## Verify and Debug

- If you want to tail the Traefik logs, you can use
```sh
$ docker service logs -f traefik
```
You can see internet traffic logs as well as logs related to the Let's Encrypt certificate process.

