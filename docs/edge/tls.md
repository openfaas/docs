# TLS for OpenFaaS Edge

The gateway is deployed using a plaintext HTTP endpoint on localhost on port 8080. It is recommended that you use TLS if the gateway is to be exposed over a public network or the Internet.

There are detailed instructions in [Serverless for Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else?layout=profile) for setting up TLS with Caddy.

* Expose the whole gateway with a custom domain
* Prevent access to certain functions
* Expose certain functions with their own custom domains

Below is a simple example to get Caddy set up and running with TLS for the OpenFaaS gateway.

## Setup Caddy for the gateway

Install Caddy via [arkade](https://arkade.dev), or download it from [caddyserver.com](https://caddyserver.com/).

```bash
curl -sLS https://get.arkade.dev | sudo sh
arkade system install caddy
```

Create a DNS A or CNAME record for the gateway using the host's public IP address - for example, `gateway.example.com`.

Then create a Caddyfile in `/var/lib/faasd`:

```caddyfile
{
  email example.com


gateway.example.com {
  reverse_proxy localhost:8080
}
```

Then restart Caddy:

```bash
sudo systemctl daemon-reload
sudo systemctl restart caddy
```

Check the logs with:

```bash
sudo journalctl -u caddy --follow
```

## Add TLS for the OpenFaaS Dashboard

If your license contains the OpenFaaS Dashboard, you can also add TLS for the dashboard.

There are three steps required:

1. Set the `public_url` environment variable in the `docker-compose.yml` file.
2. Add a DNS record for the dashboard.
3. Restart Caddy and faasd

First, edit `/var/lib/faasd/docker-compose.yml` and add the following to the `services` section:

```diff
services:
  dashboard:
    environment:
+   - "public_url=https://dashboard.example.com"
```

Create another DNS record, this time for `dashboard.example.com`, and add the following to the Caddyfile:

```caddyfile
dashboard.example.com {
  reverse_proxy localhost:8083
}
```

Then restart Caddy as before.

Then restart faasd with `sudo systemctl restart faasd`.

The dashboard will now be accessible via TLS at teh given URL i.e. `https://dashboard.example.com`.

## Expose a service

If you add a stateful service such as Grafana to the compose file, then you can use the same technique to expose it with TLS.

For instance, for Grafana, add the port to expose the service on localhost:

```yaml
  grafana:
    ports:
     - "127.0.0.1:3000:3000"
```

Then add the following to the Caddyfile:

```caddyfile
grafana.example.com {
  reverse_proxy localhost:3000
}
```
