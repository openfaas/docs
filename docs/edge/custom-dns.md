# Custom DNS for OpenFaaS Edge

By default, OpenFaaS Edge will use Google's public DNS servers to look up IP addresses from core-services and from functions. This is done to ensure that the functions can reach the Internet.

If you deploy OpenFaaS Edge within a private VPC, or enterprise network, you may need to configure custom DNS servers for functions for them to reach the Internet.

## During installation

You can specify custom DNS servers during the installation phase with:

```bash
faasd install --dns-server 1.1.1.1 --dns-server 8.8.4.4
```

## Update an existing installation

Sometimes, it's easier to update the system after the installation.

Update the systemd services for `faasd` and `faasd-provider` to include the `--dns-server` flag.

Edit the following:

* `/var/lib/faasd/faasd.service`
* `/var/lib/faasd-provider/faasd-provider.service`

In each, find the `ExecStart` line and add the `--dns-server` flag once per DNS server.
