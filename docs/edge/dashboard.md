# Dashboard for OpenFaaS Edge

The [OpenFaaS Pro Dashboard](/openfaas-pro/dashboard) is compatible with OpenFaaS Edge.

!!! note "Optional add-on"

    The dashboard is included with OpenFaaS for Kubernetes, but is an optional add-on for Edge and requires an additional tier or purchase.

## Enable the service

The service comes pre-packaged within the default installation, but is disabled by default.

Edit `/var/lib/faasd/docker-compose.yaml`:

```diff
services:
  dashboard:
-   deploy:
-     replicas: 0
```

Then create a JWT signing key:

```bash
# Generate a private key
openssl ecparam -genkey -name prime256v1 -noout -out jwt_key

# Then create a public key from the private key
openssl ec -in jwt_key -pubout -out jwt_key.pub
```

Move the keys to `/var/lib/faasd/secrets/`:

```bash
sudo mv ./jwt_key /var/lib/faasd/secrets/key
sudo mv ./jwt_key.pub /var/lib/faasd/secrets/key.pub
```

Then update the permissions for the user the dashboard runs as:

```bash
sudo chown 65534 /var/lib/faasd/secrets/key*
```

Then restart faasd:

```bash
sudo systemctl daemon-reload
sudo systemctl restart faasd
```

You can then access the dashboard from port 8083 on loopback on the host, or [add a TLS entry](/edge/tls) to access it from the Internet.

