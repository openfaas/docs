# Deploy the Kafka Connector for OpenFaaS Edge

The Kafka Connector for OpenFaaS Edge is used to trigger functions from Kafka topics.

This page covers deployment options for the connector with OpenFaaS Edge

For instructions on usage, once deployed: [see the page for OpenFaaS on Kubernetes](/openfaas-pro/kafka-events)

## Deployment modes

There are three main modes of deployment for the Kafka Connector, although some options can also be mixed such as using SASL authentication with a custom TLS bundle.

* No authentication - usually used in development, or within some enterprise networks
* SASL username and password authentication - often used with cloud-hosted Kafka brokers such as Aiven, Confluent Cloud or Amazon Managed Streaming for Apache Kafka (MSK).
* Custom TLS CA bundle - used when the Kafka broker uses a self-signed certificate or a certificate signed by a private CA.

## Environment variables

There are a number of environment variables that can be set to configure the Kafka Connector, however these are the most important:

* `topics` - the Kafka topic to listen to. This can be a comma-separated list of topics.
* `broker_hosts` - the Kafka brokers to connect to. This can be a comma-separated list of brokers.
* `upstream_timeout` - the maximum time to wait for a function to respond. This is set to 2 minutes by default.
* `rebuild_interval` - the interval to check for new functions to invoke. This is set to 30 seconds by default.
* `content_type` - the content type to use when invoking functions. This is set to `text/plain` by default.
* `group` - the Kafka consumer group to use. This is set to `faas-group-1` by default.
* `log_sessions` - whether to log sessions. This is set to `true` by default.
* `max_bytes` - the maximum number of bytes to read from the Kafka topic. This is set to 1MB by default.
* `initial_offset` - the initial offset to use when consuming messages. This is set to `oldest` by default.

## Multiple connectors

To deploy multiple connectors, give varying names to the service in the `docker-compose.yaml` file:

```yaml
kafka-connector-private:
  topics: user.signup
  broker_hosts: kafka-1:9092,kafka-2:9092,kafka-3:9092
...
kafka-connector-cloud:
  topics: user.signup
  broker_hosts: pkc-5r697.europe-west1.gcp.confluent.cloud:9092
```

## No authentication

This option uses no authentication, and turns TLS off.

It connects to three different Kafka brokers via the `broker_hosts` environment variable, and subscribes to the `user.signup` topic.

```yaml
kafka-connector:
    image: ghcr.io/openfaasltd/kafka-connector:latest
    environment:
      - gateway_url=http://gateway:8080
      - topics=user.signup
      - print_response=true
      - print_response_body=true
      - print_request_body=false
      - asynchronous_invocation=false
      - basic_auth=true
      - secret_mount_path=/run/secrets
      - broker_hosts=kafka-1:9092,kafka-2:9092,kafka-3:9092
      - upstream_timeout=2m
      - rebuild_interval=30s
      - content_type=text/plain
      - group=faas-group-1
      - log_sessions=true
      - max_bytes=1048576
      - initial_offset=oldest
    command:
      - "/usr/bin/kafka-connector"
      - "-license-file=/run/secrets/openfaas-license"
    volumes:
      # we assume cwd == /var/lib/faasd
      - type: bind
        source: ./secrets/basic-auth-password
        target: /run/secrets/basic-auth-password
      - type: bind
        source: ./secrets/basic-auth-user
        target: /run/secrets/basic-auth-user
      - type: bind
        source: "./secrets/openfaas-license"
        target: "/run/secrets/openfaas-license"
    depends_on:
      - gateway
```

## SASL authentication

The following example is for Confluent Cloud, but the same principles apply to other Kafka brokers.

TLS is enabled, however no specific CA bundle is required since Confluent Cloud uses a trust bundle already available on most systems.

Create two files in the `secrets` directory:

```bash
mkdir -p /var/lib/faasd/secrets
  echo "username" > /var/lib/faasd/secrets/broker-username
  echo "password" > /var/lib/faasd/secrets/broker-password
```

Example:

```yaml
kafka-connector:
    image: ghcr.io/openfaasltd/kafka-connector:latest
    environment:
      - gateway_url=http://gateway:8080
      - topics=user.signup
      - print_response=true
      - print_response_body=true
      - print_request_body=false
      - asynchronous_invocation=false
      - basic_auth=true
      - secret_mount_path=/run/secrets
      - broker_hosts=pkc-5r697.europe-west1.gcp.confluent.cloud:9092
      - upstream_timeout=2m
      - rebuild_interval=30s
      - content_type=text/plain
      - group=faas-group-1
      - log_sessions=true
      - max_bytes=1048576
      - initial_offset=oldest
    command:
      - "/usr/bin/kafka-connector"
      - "-license-file=/run/secrets/openfaas-license"
      - "-username-file=/run/secrets/broker-username"
      - "-password-file=/run/secrets/broker-password"
      - "-tls"
    volumes:
      # we assume cwd == /var/lib/faasd
      - type: bind
        source: ./secrets/basic-auth-password
        target: /run/secrets/basic-auth-password
      - type: bind
        source: ./secrets/basic-auth-user
        target: /run/secrets/basic-auth-user
      - type: bind
        source: "./secrets/openfaas-license"
        target: "/run/secrets/openfaas-license"
      - type: bind
        source: "./secrets/broker-username"
        target: "/run/secrets/broker-username"
      - type: bind
        source: "./secrets/broker-password"
        target: "/run/secrets/broker-password"
    depends_on:
      - gateway
```

## Custom TLS CA bundle

When a custom CA bundle is required for self-signed or untrusted certificates, the CA bundle can be mounted into the container and used by the Kafka Connector.

Create a file in the `secrets` directory:

```bash
mkdir -p /var/lib/faasd/secrets
cp ./ca-bundle.crt /var/lib/faasd/secrets/kafka-ca-bundle.crt
```

Then add the following mount:

```yaml
    volumes:
    - type: bind
      source: "./secrets/kafka-ca-bundle.crt"
      target: "/run/secrets/kafka-ca-bundle.crt"
```
    
Then add the following flag to the command:

```yaml
    command:
    - "-tls"
    - "-ca-file=/run/secrets/kafka-ca-bundle.crt"
```

## Self-signed certificate

If you want to use a self-signed certificate, which has not been signed by a CA bundle, or by a CA which is not in your trust bundle, do the following:

1. Create a self-signed certificate using OpenSSL:

```bash
openssl req -x509 -newkey rsa:2048 -keyout kafka.key -out kafka.crt -days 365 -nodes
```

2. Copy the certificate and key to the `secrets` directory:

```bash
mkdir -p /var/lib/faasd/secrets
cp kafka.crt /var/lib/faasd/secrets/kafka.crt
cp kafka.key /var/lib/faasd/secrets/kafka.key
```

Then add the following to the command:

```yaml
    command:
    - "-tls"
    - "-cert-file=/run/secrets/kafka.crt"
    - "-key-file=/run/secrets/kafka.key"
```

And then mount the two files:

```yaml
    volumes:
    - type: bind
      source: "./secrets/kafka.crt"
      target: "/run/secrets/kafka.crt"
    - type: bind
      source: "./secrets/kafka.key"
      target: "/run/secrets/kafka.key"
```

