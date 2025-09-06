# Troubleshooting OpenFaaS Edge

There are more detailed notes in the eBook.

## Find common issues

Check the services:

```bash
journalctl -u faasd
journalctl -u faasd-provider
```

Check the containers:

```bash
sudo faasd service logs gateway
sudo faasd service logs queue-worker
sudo faasd service logs faas-idler
sudo faasd service logs prometheus
```

Check service output:

```bash
sudo faasd service list
sudo faasd service top
```

Check the machine for free memory and disk space:

```bash
free -h
df -h /
```

