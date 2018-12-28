# SSL on Swarm with Traefik

To completely secure your OpenFaaS installation, you need SSL.  On Swarm, you can do this easily with [Traefik][traefik] and [Let's Encrypt][letsencrypt].  Traefik is a reverse proxy that comes with SSL support via Let's Encrypt. In this tutorial we will show you how to deploy OpenFaaS with Traefik.


## Create an A record

If your domain is `.domain.com` then create an A record using your DNS administration panel such as `gateway.domain.com` or `openfaas.domain.com`. The required steps will vary depending on your domain provider and your cluster provider. For example; [on Google Cloud DNS](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) or [with Route53 using AWS](https://kubernetes.io/docs/setup/custom-cloud/kops/#2-5-create-a-route53-domain-for-your-cluster).

## Update the Compose configuration

### Configure Traefik
To use Traefik with OpenFaaS, you need to modify the OpenFaaS deployment manifest to include Traefik and configuring OpenFaaS to communicate through Traefik instead of directly exposing its services publicly.

1. Clone OpenFaaS and then checkout the latest stable release:

    ```bash
    $ git clone https://github.com/openfaas/faas && cd faas
    ```

2. Add the `traefik` service to the  `docker-compose.yaml`.

    To start, open `docker-compose.yaml` in your favorite editor and add a `traefik` service like this:

    ```yaml
    version: "3.3"
    services:
        traefik:
            image: traefik:v1.7.6
        gateway:
            ...
    ```
3. Next, to configure Traefik to work with Docker Swarm _and_ Let's Encrypt, you must override the default `command` to the `traefik` service.  The `traefik` section of the `docker-compose.yaml` file should look like this:

    ```yaml
    ...
        traefik:
            image: traefik:v1.7.6
            command:
                - "--api=true"
                - "--docker=true"
                - "--docker.swarmmode=true"
                - "--docker.domain=traefik"
                - "--docker.watch=true"
                - "--defaultEntryPoints='http,https'"
                - "--entryPoints='Name:https Address::443 TLS'"
                - "--entryPoints='Name:http Address::80'"
                - "--acme=true"
                - "--acme.entrypoint='https'"
                - "--acme.httpchallenge=true"
                - "--acme.httpchallenge.entrypoint='http'"
                - "--acme.domains='openfaas.mydomain.com, www.openfaas.mydomain.com'"
                - "--acme.email='<your-email-here>'"
                - "--acme.ondemand=true"
                - "--acme.onhostrule=true"
                - "--acme.storage=/etc/traefik/acme/acme.json"
    ...
    ```

    * The `--api=true` flag enables Traefik's Web UI,
    * The `--docker.*` flags tell Traefik to use Docker and specify that it's running in a Docker Swarm cluster.
    * The `--defaultEntryPoints` and `--entryPoints` flags define entry points and protocols to be used. In our case this includes HTTP on port 80 and HTTPS on port 443.
    * The `--acme.*` flags configure Traefik to use the ACME protocol to generate Let's Encrypt certificates to secure your OpenFaaS cluster with SSL. Make sure to replace the `openfaas.mydomain.com` domain placeholders in the `--acme.domains` flag with your own domain. You can specify multiple domains by separating them with a comma and space.
4. Expose the ports required for Traefik.  Traefik uses port `8080` for its operations and UI, while in the last step we configured ports `80` and `443` and entrypoints for internet traffic.  These will be used / proxied to OpenFaaS

    ```yaml
        ...
        traefik:
            image: traefik:v1.7.6
            command:
                ...
            ports:
                - 80:80
                - 8080:8080
                - 443:443
        ...
    ```
5. Next you must configure the volumes needed for Traefik, in this case, the docker socket and some file storage. The Docker socket file communicates with the Docker API in order to manage your containers and will provide details about them, such as number of containers and their IP addresses, to Traefik. You will also mount the volume called acme, which we'll define later in this step.

    ```yaml
        ...
        traefik:
            image: traefik:v1.7.6
            command:
                ...
            ports:
                ...
            volumes:
                - "/var/run/docker.sock:/var/run/docker.sock"
                - "acme:/etc/traefik/acme"
        ...
    ```
6. Next you must configure the networks the `traefik` service is part of. All OpenFaaS components live on the `functions` networks, which is also defined in the compose file.

    ```yaml
        ...
        traefik:
            image: traefik:v1.7.6
            command:
                ...
            ports:
                ...
            volumes:
                ...
            networks:
                - functions
        ...
    ```
7. Lastly, you must configure the `deploy` section so that Traefik is only deployed on the Docker Swarm manager.

    ```yaml
        ...
        traefik:
            image: traefik:v1.7.6
            command:
                ...
            ports:
                ...
            volumes:
                ...
            networks:
                ...
            deploy:
                placement:
                    constraints: [node.role == manager]
        ...
    ```

All together, the `traefik` service block should look like

```yaml
version: "3.3"
services:
    traefik:
        image: traefik:v1.7.6
        command:
            - "--api=true"
            - "--docker=true"
            - "--docker.swarmmode=true"
            - "--docker.domain=traefik"
            - "--docker.watch=true"
            - "--defaultEntryPoints='http,https'"
            - "--entryPoints='Name:https Address::443 TLS'"
            - "--entryPoints='Name:http Address::80'"
            - "--acme=true"
            - "--acme.entrypoint='https'"
            - "--acme.httpchallenge=true"
            - "--acme.httpchallenge.entrypoint='http'"
            - "--acme.domains='openfaas.mydomain.com, www.openfaas.mydomain.com'"
            - "--acme.email='<your-email-here>'"
            - "--acme.ondemand=true"
            - "--acme.onhostrule=true"
            - "--acme.storage=/etc/traefik/acme/acme.json"
        ports:
            - 80:80
            - 8080:8080
            - 443:443
        volumes:
            - "/var/run/docker.sock:/var/run/docker.sock"
            - "acme:/etc/traefik/acme"
        networks:
          - functions
        deploy:
          placement:
            constraints: [node.role == manager]

    gateway:
        ...
```

### Configure OpenFaaS
By default, the original `docker-compose.yaml` file exposes the OpenFaaS `gateway` on port `8080`.  This conflicts with the configuration of Traefik, additionally, we want all communication to safely pass through Traefik.  To do this you must make two modifications to the `gateway` service.

First, remove the `ports` section.

Next, add the following `labels` directive to the `deploy` section of the gateway service.

```yaml
    gateway:
        image: ...
        networks:
            ...
        environment:
            ...
        deploy:
            labels:
                - traefik.port=8080
                - traefik.frontend.rule=PathPrefix:/ui,/system,/function
            resources:
                ...
        secrets:
            ...
    ...
```

These labels expose the OpenFaaS gateway `/ui`, `/system`, and `/function` endpoints on port `8080` over Traefik.

All together, your `gateway` service should now look like

```yaml
...
    gateway:
        image: openfaas/gateway:0.9.11
        networks:
            - functions
        environment:
            functions_provider_url: "http://faas-swarm:8080/"
            read_timeout:  "300s"        # Maximum time to read HTTP request
            write_timeout: "300s"        # Maximum time to write HTTP response
            upstream_timeout: "300s"     # Maximum duration of upstream function call - should be more than read_timeout and write_timeout
            dnsrr: "true"               # Temporarily use dnsrr in place of VIP while issue persists on PWD
            faas_nats_address: "nats"
            faas_nats_port: 4222
            direct_functions: "true"    # Functions are invoked directly over the overlay network
            direct_functions_suffix: ""
            basic_auth: "${BASIC_AUTH:-true}"
            secret_mount_path: "/run/secrets/"
            scale_from_zero: "true"
        deploy:
            labels:
                - traefik.port=8080
                - traefik.frontend.rule=PathPrefix:/ui,/system,/function
            resources:
                # limits:   # Enable if you want to limit memory usage
                #     memory: 200M
                reservations:
                    memory: 100M
            restart_policy:
                condition: on-failure
                delay: 5s
                max_attempts: 20
                window: 380s
            placement:
                constraints:
                    - 'node.platform.os == linux'
        secrets:
            - basic-auth-user
            - basic-auth-password
...
```

### Configure data volumes

Finally, while configuring Traefik, you mounted a volume called `acme`.  You must now define the `acme` volume used for storing Let's Encrypt certificates. We can define an empty volume, meaning data will not persist if you destroy the container. If you destroy the container, the certificates will be regenerated the next time you start Traefik.

Add the following volumes directive on the last line of the file:

```yaml
...
volumes:
    acme:
```

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
faas-cli login --gateway https://openfaas.mydomain.com --username admin --password <randomly-generated-password>
faas-cli deploy --gateway https://openfaas.mydomain.com
```
Replace `openfaas.mydomain.com` with your domain as well as adding the username `admin` and secure random password that the deploy script created for you when you deployed OpenFaaS.

### Using the web UI
You can use the web UI to see the functions deployed to your cluster or to deploy functions from the Store.  In your web browser, go to https://openfaas.mydomain.com/ui/. Note that the trailing slash is required.

On your first visit, the HTTP authentication dialogue box will open, you can login with the username `admin` and secure random password that the deploy script created for you when you deployed OpenFaaS.

## Verify and Debug

- If you want to tail the Traefik logs, you can use
```sh
$ docker service ls -f traefik
```
You can see internet traffic logs as well as logs related to the Let's Encrypt certificate process.

## Profit!

[traefik]: https://traefik.io/
[letsencrypt]: https://letsencrypt.org/
