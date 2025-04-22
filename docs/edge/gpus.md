# GPU support for OpenFaaS Edge

At the time of speaking, OpenFaaS Pro on Kubernetes supports GPUs for functions via Profiles.

OpenFaaS Edge supports Nvidia GPUs for core services only.

Example:

```yaml
services:  
  ollama:
    image: docker.io/ollama/ollama:latest
    command:
      - "ollama"
      - "serve"
    volumes:
      - type: bind
        source: ./ollama
        target: /root/.ollama
    ports:
      - "127.0.0.1:11434:11434"
    gpus: all
    deploy:
      restart: always
```

Learn more: [How to Protect Your Data with Self-Hosted LLMs and OpenFaaS Edge](https://www.openfaas.com/blog/local-llm-openfaas-edge/)

