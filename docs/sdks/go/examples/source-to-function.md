Use the [OpenFaaS Go SDK](https://github.com/openfaas/go-sdk) and the [Function Builder API](/openfaas-pro/builder/) to go from source code to a running, authenticated function in a custom namespace, entirely from Go.

Use-cases:

* SaaS platforms where users supply their own code
* Multi-tenant environments where each tenant gets an isolated namespace
* CI/CD pipelines that go from source to a running function in a single program

The `greeter` function validates a Bearer token against a mounted secret. The orchestration program creates a namespace and secret, builds the function image from source, deploys it, and invokes the function once it is ready.

## Overview

The example consists of a `greeter` function and an orchestration program (`main.go`) that drives the full workflow.

greeter/handler.go:

```go
package function

import (
	"encoding/json"
	"fmt"
	"net/http"
	"os"
)

func Handle(w http.ResponseWriter, r *http.Request) {
	apiKey, err := os.ReadFile("/var/openfaas/secrets/api-key")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: secret not available: %v\n", err)
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusInternalServerError)
		json.NewEncoder(w).Encode(map[string]string{"error": "secret not available"})
		return
	}

	authHeader := r.Header.Get("Authorization")
	if len(authHeader) < 8 || authHeader[:7] != "Bearer " {
		fmt.Fprintln(os.Stderr, "Unauthorized: missing or malformed Authorization header")
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusUnauthorized)
		json.NewEncoder(w).Encode(map[string]string{"error": "Unauthorized"})
		return
	}

	if authHeader[7:] != string(apiKey) {
		fmt.Fprintln(os.Stderr, "Unauthorized: invalid token")
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusUnauthorized)
		json.NewEncoder(w).Encode(map[string]string{"error": "Unauthorized"})
		return
	}

	fmt.Println("Request authorized, returning greeting")
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"message": "Hello from OpenFaaS!"})
}
```

main.go:

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strings"
	"time"

	sdk "github.com/openfaas/go-sdk"
	"github.com/openfaas/go-sdk/builder"
	"github.com/openfaas/faas-provider/types"
)

const (
	namespace    = "tenant1"
	functionName = "greeter"
	secretName   = "api-key"
	image        = "ttl.sh/openfaas/greeter:1h"
)

func main() {
	gatewayURL, err := url.Parse(os.Getenv("OPENFAAS_GATEWAY"))
	if err != nil || gatewayURL.Host == "" {
		log.Fatal("OPENFAAS_GATEWAY is required")
	}
	builderURL, err := url.Parse(os.Getenv("BUILDER_URL"))
	if err != nil || builderURL.Host == "" {
		log.Fatal("BUILDER_URL is required")
	}
	password := os.Getenv("OPENFAAS_PASSWORD")
	if password == "" {
		log.Fatal("OPENFAAS_PASSWORD is required")
	}
	payloadSecretBytes, err := os.ReadFile(os.Getenv("PAYLOAD_SECRET_PATH"))
	if err != nil {
		log.Fatalf("read payload secret: %v", err)
	}
	payloadSecret := strings.TrimSpace(string(payloadSecretBytes))

	auth := &sdk.BasicAuth{Username: "admin", Password: password}
	client := sdk.NewClient(gatewayURL, auth, http.DefaultClient)

	ctx := context.Background()

	// Create an isolated namespace for the tenant.
	log.Printf("Creating namespace %q", namespace)
	if _, err := client.CreateNamespace(ctx, types.FunctionNamespace{Name: namespace}); err != nil {
		log.Fatalf("create namespace: %v", err)
	}

	// Generate a random API key and store it as an OpenFaaS secret.
	// The function reads this value at runtime from the mounted secret file.
	apiKey := fmt.Sprintf("%d", time.Now().UnixNano())
	log.Printf("Creating secret %q", secretName)
	if _, err := client.CreateSecret(ctx, types.Secret{
		Name:      secretName,
		Namespace: namespace,
		Value:     apiKey,
	}); err != nil {
		log.Fatalf("create secret: %v", err)
	}

	// Assemble the Docker build context from the template and handler,
	// then pack it into a tar archive with the build configuration.
	log.Printf("Assembling build context for %q", functionName)
	contextPath, err := builder.CreateBuildContext(functionName, "./greeter", "golang-middleware", []string{},
		builder.WithTemplateDir("./template"),
		builder.WithBuildDir("/tmp/build"),
	)
	if err != nil {
		log.Fatalf("create build context: %v", err)
	}

	tarPath := "/tmp/greeter.tar"
	if err := builder.MakeTar(tarPath, contextPath, &builder.BuildConfig{
		Image:     image,
		Platforms: []string{"linux/amd64"},
	}); err != nil {
		log.Fatalf("make tar: %v", err)
	}

	// Send the tar to the Function Builder and stream log lines as they arrive.
	log.Printf("Building %q", image)
	b := builder.NewFunctionBuilder(builderURL, http.DefaultClient, builder.WithHmacAuth(payloadSecret))
	stream, err := b.BuildWithStream(tarPath)
	if err != nil {
		log.Fatalf("build: %v", err)
	}
	defer stream.Close()

	for result, err := range stream.Results() {
		if err != nil {
			log.Fatalf("build stream: %v", err)
		}
		for _, line := range result.Log {
			fmt.Println(line)
		}
		if result.Status == builder.BuildFailed {
			log.Fatalf("build failed: %s", result.Error)
		}
	}

	// Deploy the function into the tenant namespace with the secret mounted.
	log.Printf("Deploying %q into %q", functionName, namespace)
	if _, err := client.Deploy(ctx, types.FunctionDeployment{
		Service:   functionName,
		Image:     image,
		Namespace: namespace,
		Secrets:   []string{secretName},
	}); err != nil {
		log.Fatalf("deploy: %v", err)
	}

	log.Println("Waiting for function to become ready")
	if err := waitForReady(ctx, client, functionName, namespace, 120*time.Second); err != nil {
		log.Fatal(err)
	}

	// Invoke the function with the generated API key as a Bearer token.
	log.Println("Invoking function with valid token")
	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, "/", nil)
	req.Header.Set("Authorization", "Bearer "+apiKey)
	res, err := client.InvokeFunction(functionName, namespace, false, false, req)
	if err != nil {
		log.Fatalf("invoke: %v", err)
	}
	body, _ := io.ReadAll(res.Body)
	res.Body.Close()
	fmt.Printf("%d %s\n", res.StatusCode, body)

	// Invoke with a bad token to verify 401.
	log.Println("Invoking function with invalid token")
	req, _ = http.NewRequestWithContext(ctx, http.MethodGet, "/", nil)
	req.Header.Set("Authorization", "Bearer wrong-token")
	res, err = client.InvokeFunction(functionName, namespace, false, false, req)
	if err != nil {
		log.Fatalf("invoke: %v", err)
	}
	body, _ = io.ReadAll(res.Body)
	res.Body.Close()
	fmt.Printf("%d %s\n", res.StatusCode, body)

	// Get the last 20 log lines from the function.
	log.Println("Fetching logs")
	tail := 20
	logCh, err := client.GetLogs(ctx, functionName, namespace, false, tail, nil)
	if err != nil {
		log.Fatalf("get logs: %v", err)
	}
	for msg := range logCh {
		fmt.Printf("[%s] %s: %s\n", msg.Timestamp, msg.Instance, msg.Text)
	}
}

func waitForReady(ctx context.Context, client *sdk.Client, name, namespace string, timeout time.Duration) error {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	ticker := time.NewTicker(3 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return fmt.Errorf("timed out waiting for %q to become ready", name)
		case <-ticker.C:
			fn, err := client.GetFunction(ctx, name, namespace)
			if err == nil && fn.AvailableReplicas >= 1 {
				return nil
			}
		}
	}
}
```

- A timestamp-based value is used as the API key and stored as an OpenFaaS secret.
- `CreateBuildContext` assembles a Docker build context from the OpenFaaS template and handler directory. `MakeTar` packs it with the build configuration into a tar archive ready for the Function Builder. `BuildWithStream` sends the tar and yields log lines as they arrive.

## Step-by-step walkthrough

### Prerequisites

- Go 1.21+
- [`faas-cli`](https://github.com/openfaas/faas-cli) installed
- OpenFaaS with the [Function Builder](/openfaas-pro/builder/) enabled
- A container registry the builder can push to. This example uses [ttl.sh](https://ttl.sh) (no credentials required)

### Set up the project

Create a directory for the example:

```bash
mkdir source-to-function && cd source-to-function
```

Pull the `golang-middleware` template:

```bash
faas-cli template store pull golang-middleware
```

Scaffold the `greeter` function handler:

```bash
faas-cli new --lang golang-middleware greeter
```

Replace `greeter/handler.go` with the handler from the overview.

Initialise the Go module and add the SDK dependency:

```bash
go mod init source-to-function
go get github.com/openfaas/go-sdk
```

Create `main.go` in the project root with the script from the overview.

### Configure environment variables

| Variable | Default | Description |
|---|---|---|
| `OPENFAAS_PASSWORD` | — | **Required.** OpenFaaS gateway admin password |
| `OPENFAAS_GATEWAY` | — | **Required.** Gateway URL |
| `BUILDER_URL` | — | **Required.** Function Builder URL |
| `PAYLOAD_SECRET_PATH` | — | **Required.** Path to the builder HMAC payload secret |

Retrieve the gateway password and builder payload secret from the cluster:

```bash
export OPENFAAS_PASSWORD=$(kubectl get secret -n openfaas basic-auth \
  -o jsonpath="{.data.basic-auth-password}" | base64 --decode)

kubectl get secret -n openfaas payload-secret \
  -o jsonpath='{.data.payload-secret}' | base64 --decode \
  > /tmp/payload-secret
```

### Run the program

```bash
export OPENFAAS_GATEWAY=https://gateway.example.com
export BUILDER_URL=https://builder.example.com
export PAYLOAD_SECRET_PATH=/tmp/payload-secret

go run main.go
```
