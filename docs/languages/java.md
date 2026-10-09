## Java

The `java21-vertx` template is recommended for Java functions. It uses Java 21 LTS,
[Vert.x](https://vertx.io/) and Gradle, with the of-watchdog handling HTTP requests.

!!! info "Do you need to customise this template?"

    You can customise the official template, or provide your own. The code is available on GitHub: [openfaas/templates](https://github.com/openfaas/templates).

## Create a new function

Pull the default templates:

```bash
faas-cli template pull
```

Scaffold a function with a name of your choice:

```bash
faas-cli new --lang java21-vertx my-function
```

The function directory contains:

```text
my-function/
├── build.gradle
└── src/main/java/com/openfaas/function/Handler.java
```

`build.gradle` defines your dependencies. Add your function logic to
`src/main/java/com/openfaas/function/Handler.java`.

The default handler returns a greeting using the request body:

```java
package com.openfaas.function;

import io.vertx.core.Vertx;
import io.vertx.ext.web.RoutingContext;

public class Handler implements io.vertx.core.Handler<RoutingContext> {
    private final Vertx vertx;

    public Handler(Vertx vertx) {
        this.vertx = vertx;
    }

    @Override
    public void handle(RoutingContext context) {
        String body = context.body().asString();
        String name = body == null || body.isBlank() ? "World" : body;
        context.response()
            .putHeader("Content-Type", "text/plain; charset=utf-8")
            .end("Hello, " + name + "!");
    }
}
```

The template provides the HTTP server and calls `handle` for each request.
The `Vertx` instance is passed to the handler's constructor.

## Working with requests and JSON

Use the handler's `RoutingContext` to read request data and return a response.
This example returns the method, path, query parameter, header and body as JSON.

Add `import io.vertx.core.json.JsonObject;` to the handler and update the `handle` method:

```java
@Override
public void handle(RoutingContext context) {
    JsonObject body = new JsonObject()
        .put("method", context.request().method().name())
        .put("path", context.request().path())
        .put("name", context.queryParam("name").stream()
            .findFirst().orElse("World"))
        .put("requestId", context.request().getHeader("X-Request-Id"))
        .put("body", context.body().asString());

    context.response()
        .setStatusCode(200)
        .putHeader("Content-Type", "application/json; charset=utf-8")
        .end(body.encode());
}
```

Deploy the function, then invoke it with a path and query parameter such as
`/greet?name=OpenFaaS`.

## Reuse clients between invocations

The template creates one handler per process. Initialise shared clients in the
constructor and keep request data in local variables or the `RoutingContext`.

Use asynchronous clients to avoid blocking other requests. See the
[HTTP client](#make-an-http-request-to-another-function) and
[database connection pool](#access-a-sql-database) examples below.

## Add Maven dependencies

Add libraries to the `dependencies` block in `build.gradle`.
Maven Central and the Vert.x platform are already configured, so Vert.x modules
do not need a version.

For example, add the Vert.x Web Client to make outgoing HTTP requests:

```groovy
implementation 'io.vertx:vertx-web-client'
```

For other Maven libraries, use `implementation 'group:artifact:version'`.
To include a JAR, place it under `libs/` and add:

```groovy
implementation files('libs/helper.jar')
```

## Add Java packages

Place your own classes under `src/main/java/`, in directories matching
their package. Gradle includes these classes automatically, so no changes to
`build.gradle` are needed.

For example, create `src/main/java/com/openfaas/function/util/Greeting.java`:

```java
package com.openfaas.function.util;

public class Greeting {
    public static String greet(String name) {
        return "Hello, " + name + "!";
    }
}
```

Add `import com.openfaas.function.util.Greeting;` to `Handler.java`, then use
the class in your `handle` method:

```java
@Override
public void handle(RoutingContext context) {
    context.response().end(Greeting.greet("OpenFaaS"));
}
```

## Unit testing

JUnit Jupiter is already configured. Add tests under `src/test/java/`, they run
automatically during the image build.

## Make an HTTP request to another function

This example calls the `nodeinfo` store function through the OpenFaaS gateway.
Make sure the upstream function is deployed:

```bash
faas-cli store deploy nodeinfo
```

Add the Web Client to the existing `dependencies` block in
`build.gradle`:

```groovy
implementation 'io.vertx:vertx-web-client'
```

Create a shared http client:

```java
package com.openfaas.function;

import io.vertx.core.Vertx;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.web.RoutingContext;
import io.vertx.ext.web.client.WebClient;

public class Handler implements io.vertx.core.Handler<RoutingContext> {
    private final WebClient client;
    private final String upstreamUrl;

    public Handler(Vertx vertx) {
        client = WebClient.create(vertx);

        upstreamUrl = System.getenv().getOrDefault(
            "UPSTREAM_URL",
            "http://gateway.openfaas:8080/function/nodeinfo"
        );
    }

    @Override
    public void handle(RoutingContext context) {
        if (!context.request().method().name().equals("GET")) {
            context.response().putHeader("Allow", "GET");
            respond(context, 405, new JsonObject().put("error", "Use GET"));
            return;
        }

        // Send asynchronously to keep the event loop free.
        client.getAbs(upstreamUrl)
            .followRedirects(false)
            .timeout(3000)
            .send()
            .onSuccess(response -> {
                // HTTP errors also reach onSuccess; check the status code.
                if (response.statusCode() < 200
                    || response.statusCode() >= 300) {
                    JsonObject body = new JsonObject()
                        .put("error", "Upstream rejected the request")
                        .put("upstreamStatus", response.statusCode());
                    respond(context, 502, body);
                    return;
                }

                JsonObject body = new JsonObject()
                    .put("upstreamStatus", response.statusCode())
                    .put("body", response.bodyAsString());
                respond(context, 200, body);
            })
            .onFailure(error -> {
                JsonObject record = new JsonObject()
                    .put("event", "upstream_request_failed")
                    .put("errorType", error.getClass().getSimpleName());
                System.err.println(record.encode());
                respond(context, 502, new JsonObject()
                    .put("error", "Upstream unavailable or timed out"));
            });
    }

    private static void respond(
        RoutingContext context, int status, JsonObject body
    ) {
        context.response()
            .setStatusCode(status)
            .putHeader("Content-Type", "application/json; charset=utf-8")
            .end(body.encode());
    }
}
```

The client is reused between invocations. Requests run asynchronously, allowing
the function to handle other invocations while waiting for the upstream response.

## Access a SQL database

This example uses PostgreSQL to query a SQL database with a shared connection pool.

Add the PostgreSQL client to the existing `dependencies` block in
`build.gradle`:

```groovy
implementation 'io.vertx:vertx-pg-client'
```

Connection settings use environment variables, while the password is read from
an [OpenFaaS secret](/reference/secrets/). The function's configuration in
`stack.yaml` might look like this:

```yaml
functions:
  my-function:
    lang: java21-vertx
    handler: ./my-function
    image: my-function:latest
    environment:
      PGHOST: postgres.java-examples-db.svc.cluster.local
      PGPORT: "5432"
      PGUSER: app_user
      PGDATABASE: app_db
    secrets:
      - postgres-password
```

This example reuses a connection pool and reads the password from the mounted
secret. Queries run asynchronously, so other requests can be handled while
waiting for the database:

```java
package com.openfaas.function;

import io.vertx.core.Vertx;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.web.RoutingContext;
import io.vertx.pgclient.PgBuilder;
import io.vertx.pgclient.PgConnectOptions;
import io.vertx.sqlclient.Pool;
import io.vertx.sqlclient.PoolOptions;
import io.vertx.sqlclient.Tuple;

import java.util.concurrent.TimeUnit;

public class Handler implements io.vertx.core.Handler<RoutingContext> {
    private static final String SECRET_PATH =
        "/var/openfaas/secrets/postgres-password";

    private final Pool pool;

    public Handler(Vertx vertx) {
        PoolOptions options = new PoolOptions()
            .setMaxSize(4)
            .setMaxWaitQueueSize(32)
            .setConnectionTimeout(5)
            .setConnectionTimeoutUnit(TimeUnit.SECONDS);

        pool = PgBuilder.pool()
            .using(vertx)
            .with(options)
            // Read the secret when opening a connection.
            .connectingTo(() -> vertx.fileSystem()
                .readFile(SECRET_PATH)
                .map(buffer -> new PgConnectOptions()
                    .setHost(env("PGHOST",
                        "postgres.java-examples-db.svc.cluster.local"))
                    .setPort(Integer.parseInt(env("PGPORT", "5432")))
                    .setDatabase(env("PGDATABASE", "app_db"))
                    .setUser(env("PGUSER", "app_user"))
                    .setPassword(buffer.toString().strip())))
            .build();
    }

    @Override
    public void handle(RoutingContext context) {
        if (!context.request().method().name().equals("GET")) {
            context.response().putHeader("Allow", "GET");
            respond(context, 405, new JsonObject().put("error", "Use GET"));
            return;
        }

        String name = context.queryParam("name").stream()
            .findFirst().orElse("OpenFaaS");
        if (name.length() > 100) {
            respond(context, 400, new JsonObject()
                .put("error", "name must be at most 100 characters"));
            return;
        }

        String query =
            "SELECT $1::text AS name, current_database() AS database, "
            + "pg_backend_pid() AS connection_id";

        pool.preparedQuery(query)
            .execute(Tuple.of(name))
            .onSuccess(rows -> {
                var row = rows.iterator().next();
                JsonObject body = new JsonObject()
                    .put("message", "Hello, " + row.getString("name") + "!")
                    .put("database", row.getString("database"))
                    .put("connectionId", row.getInteger("connection_id"));
                respond(context, 200, body);
            })
            .onFailure(error -> {
                JsonObject record = new JsonObject()
                    .put("event", "database_query_failed")
                    .put("errorType", error.getClass().getSimpleName());
                System.err.println(record.encode());
                respond(context, 503, new JsonObject()
                    .put("error", "Database unavailable"));
            });
    }

    private static String env(String name, String fallback) {
        return System.getenv().getOrDefault(name, fallback);
    }

    private static void respond(
        RoutingContext context, int status, JsonObject body
    ) {
        context.response()
            .setStatusCode(status)
            .putHeader("Content-Type", "application/json; charset=utf-8")
            .end(body.encode());
    }
}
```

## Include and serve static files

Functions can serve static content such as HTML pages, stylesheets or JSON files
alongside dynamic responses.

Place files under `src/main/resources/webroot/`. Gradle includes
them in the function JAR automatically:

```text
src/main/resources/webroot/
├── index.html
├── style.css
└── message.json
```

Use Vert.x's `StaticHandler` to serve the packaged files through your function:

```java
package com.openfaas.function;

import io.vertx.core.Vertx;
import io.vertx.ext.web.Router;
import io.vertx.ext.web.RoutingContext;
import io.vertx.ext.web.handler.StaticHandler;

public class Handler implements io.vertx.core.Handler<RoutingContext> {
    private final Router router;

    public Handler(Vertx vertx) {
        router = Router.router(vertx);

        // Serve packaged resources; / defaults to index.html.
        StaticHandler files = StaticHandler.create("webroot")
            .setIncludeHidden(false)
            .setDirectoryListing(false)
            .setAlwaysAsyncFS(true)
            .setMaxAgeSeconds(60);

        router.route().handler(files);

        // Missing files fall through to this route.
        router.route().handler(context -> context.response()
            .setStatusCode(404)
            .putHeader("Content-Type", "text/plain; charset=utf-8")
            .end("Not found\n"));
    }

    @Override
    public void handle(RoutingContext context) {
        router.handleContext(context);
    }
}
```

The router maps request paths to packaged files: `/` serves `index.html`, and
`/style.css` serves the stylesheet.

## Authenticate requests with middleware

Use middleware to authenticate requests with a shared bearer token. This example
protects every path and HTTP method without additional dependencies.

A random token can be generated with OpenSSL:

```bash
openssl rand -hex 32
```

The token is stored in an [OpenFaaS secret](/reference/secrets/) named
`auth-bearer`, referenced in the function's `stack.yaml` configuration:

```yaml
functions:
  my-function:
    lang: java21-vertx
    handler: ./my-function
    image: my-function:latest
    secrets:
      - auth-bearer
```

The handler reads the token from the mounted secret at startup and reuses it
between invocations. If the secret changes, restart or redeploy the function
to load the new value.

```java
package com.openfaas.function;

import io.vertx.core.Vertx;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.web.Router;
import io.vertx.ext.web.RoutingContext;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;

public class Handler implements io.vertx.core.Handler<RoutingContext> {
    private static final String SECRET_PATH =
        "/var/openfaas/secrets/auth-bearer";

    private final byte[] bearerToken;
    private final Router router;

    public Handler(Vertx vertx) {
        bearerToken = readBearerToken(vertx);
        router = Router.router(vertx);

        // Authenticate every request before running the handler.
        router.route()
            .handler(this::authenticate)
            .handler(context -> {
                JsonObject body = new JsonObject()
                    .put("message", "Authenticated request");
                respond(context, 200, body);
            });
    }

    @Override
    public void handle(RoutingContext context) {
        router.handleContext(context);
    }

    private static byte[] readBearerToken(Vertx vertx) {
        // Read once during startup, before handling requests.
        String token = vertx.fileSystem()
            .readFileBlocking(SECRET_PATH)
            .toString(StandardCharsets.UTF_8)
            .strip();

        if (token.isEmpty()) {
            throw new IllegalStateException(
                "OpenFaaS secret auth-bearer must not be empty");
        }

        return token.getBytes(StandardCharsets.UTF_8);
    }

    private void authenticate(RoutingContext context) {
        String authorization = context.request().getHeader("Authorization");
        if (authorization == null
            || !authorization.regionMatches(true, 0, "Bearer ", 0, 7)) {
            unauthorized(context);
            return;
        }

        byte[] suppliedToken = authorization.substring(7)
            .getBytes(StandardCharsets.UTF_8);
        // Compare tokens without an early exit on the first mismatch.
        if (!MessageDigest.isEqual(bearerToken, suppliedToken)) {
            unauthorized(context);
            return;
        }

        context.next();
    }

    private static void unauthorized(RoutingContext context) {
        context.response()
            .putHeader("WWW-Authenticate", "Bearer realm=\"private-api\"");
        respond(context, 401, new JsonObject()
            .put("error", "A valid bearer token is required"));
    }

    private static void respond(
        RoutingContext context, int status, JsonObject body
    ) {
        context.response()
            .setStatusCode(status)
            .putHeader("Content-Type", "application/json; charset=utf-8")
            .putHeader("Cache-Control", "no-store")
            .end(body.encode());
    }
}
```

Callers send the token in the `Authorization: Bearer <token>` header. The
middleware checks it before passing the request to the handler with `next()`.
Requests with a missing or invalid token are rejected.

OpenFaaS also supports authentication without changing your handler code. See
[OpenFaaS IAM function authentication](/openfaas-pro/iam/function-authentication/)
for access control with roles and policies, or
[OAuth/OIDC login](/reference/function-oauth/) for browser sign-in.
