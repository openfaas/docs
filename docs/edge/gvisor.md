# gVisor runtime

Improve container security with [gVisor](https://gvisor.dev/). The gVisor runsc runtime improves isolation between the Linux host and function containers so you can safely run untrusted code e.g. user-uploaded, LLM-generated, or third-party code. 

OpenFaaS Edge supports using the gVisor runsc runtime for functions.
If you are using OpenFaaS Pro on Kubernetes the runtime is supported via [Profiles](/reference/profiles/#use-an-alternative-runtimeclass). 

## Installation

To start using gVisor with OpenFaaS Edge install runsc and the containerd runsc shim using the [gVisor installation docs](https://gvisor.dev/docs/user_guide/install/).

> Note: The containerd configuration does not need to be updated to use gVisor with OpenFaaS Edge.

### New OpenFaaS Edge installation

[Follow the installation instructions](/deployment/edge/#openfaas-edgefaasd-pro-commercial-use) to install OpenFaaS Edge. When you reach the step to run the `faasd install` command make sure to add the `--gvisor` flag:

```sh
faasd install --gvisor
```

### Change the runtime for an existing installation

When you want to change the runtime for an existing OpenFaaS Edge deployment run:

```sh
faasd install --gvisor

systemctl daemon-reload
systemctl restart faasd-provider
```

Make sure to redeploy functions to switch them over to the new runtime.
