The Fan-in pattern complements the
[Fan-out pattern](/languages/patterns/#fan-out-pattern) by bringing the
independent results back together. The workflow can then continue after every
item in the batch has completed.

A Fan-in implementation associates each result with its batch, records
progress as results arrive, and triggers the next step once all expected work
is complete. It should also store the individual results when the next step needs
to combine them.

This example extends the [Fan-out example](/languages/patterns/fan-out/) by
sending the result of each `batch-worker` invocation to a `fan-in` function.
The function uses a PostgreSQL counter to track completion and invokes the
next function when the counter reaches zero.

```text
1. Initialize the batch

[ Client ] ── size 3 ──► [ fan-in ] ──► [ PostgreSQL: remaining 3 ]
    ▲                         │
    └────── batch ID ─────────┘

2. Fan the callbacks back in

[ Queue-worker ]
      ├── "one" ───► [ batch-worker ] ──┐
      ├── "two" ───► [ batch-worker ] ──┼── callbacks ──► [ fan-in ]
      └── "three" ─► [ batch-worker ] ──┘                      │
                                                               ▼
                                               [ PostgreSQL: 3 → 2 → 1 → 0 ]
                                                               │
                                                               ▼
                                                           [ printer ]
```

* A counter is created with the number of items before any
  work is submitted.
* Every worker result is sent to the same callback function
  with the batch ID.
* PostgreSQL updates the counter atomically, so callbacks
  can arrive concurrently and reach different `fan-in` replicas.
* Only the callback that changes the counter to zero invokes
  the next function for the completed batch.

The example only tracks completion. A workflow that needs to combine results
would also store each callback body under the batch ID, then have the final
function read and combine those results.

## Create the function

The `fan-out` and `batch-worker` functions come from the
[Fan-out example](/languages/patterns/fan-out/). Scaffold the additional `fan-in` function:

=== "Go"

    ```bash
    mkdir fan-in && cd fan-in
    faas-cli template store pull golang-middleware
    faas-cli new --lang golang-middleware fan-in \
      --prefix ttl.sh/openfaas-examples
    ```

=== "Python"

    ```bash
    mkdir fan-in && cd fan-in
    faas-cli template store pull python3-http
    faas-cli new --lang python3-http fan-in \
      --prefix ttl.sh/openfaas-examples
    ```

=== "Node.js"

    ```bash
    mkdir fan-in && cd fan-in
    faas-cli template store pull node24
    faas-cli new --lang node24 fan-in \
      --prefix ttl.sh/openfaas-examples
    ```

The example uses the public [ttl.sh](https://ttl.sh) registry. Replace the
prefix with your own registry for production use.

The full source code and `stack.yaml` files are available on GitHub for
[Go](https://github.com/openfaas/function-patterns/tree/master/go/fan-in),
[Python](https://github.com/openfaas/function-patterns/tree/master/python/fan-in),
and [Node.js](https://github.com/openfaas/function-patterns/tree/master/node/fan-in).

## Configure PostgreSQL

Create a table with one row for each batch. Completed rows remain at zero so a
late callback cannot invoke the final function again.

`fan-in/schema.sql`:

```sql
CREATE TABLE IF NOT EXISTS batch_counters (
  batch_id text PRIMARY KEY,
  remaining integer NOT NULL CHECK (remaining >= 0)
);
```

Apply the schema to a PostgreSQL database that the function can reach:

```bash
psql "$POSTGRES_CONNECTION" -f fan-in/schema.sql
```

Store the same connection string in `postgres-connection.txt`, then create an
OpenFaaS secret from it:

```bash
faas-cli secret create postgres-connection \
  --from-file=postgres-connection.txt
```

The function reads the connection string from
`/var/openfaas/secrets/postgres-connection`.

## Implement fan-in

The function exposes two paths:

* `POST /batch` accepts a size, creates the counter, and returns a generated
  batch ID.
* `POST /callback?batch_id=<id>` atomically decrements the counter. The
  callback that changes it to zero posts a completion message to `notify_url`.

The SQL statement used by `/callback` both updates and reads the counter. The
`decremented` value distinguishes the callback that completed the batch from a
later callback that reads the existing zero.

=== "Go"

    `fan-in/handler.go`:

    ```go
    package function

    import (
        "bytes"
        "context"
        "encoding/json"
        "errors"
        "fmt"
        "net/http"
        "os"
        "strings"
        "time"

        "github.com/jackc/pgx/v5"
    )

    const (
        connectionSecret = "/var/openfaas/secrets/postgres-connection"
        notifyTimeout     = 5 * time.Second
    )

    const decrementQuery = `
    WITH updated AS (
        UPDATE batch_counters
           SET remaining = remaining - 1
         WHERE batch_id = $1
           AND remaining > 0
     RETURNING remaining
    )
    SELECT remaining, true
      FROM updated
    UNION ALL
    SELECT remaining, false
      FROM batch_counters
     WHERE batch_id = $1
       AND NOT EXISTS (SELECT 1 FROM updated)
    LIMIT 1`

    type postgresCounter struct {
        conn *pgx.Conn
    }

    func Handle(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()

        connectionString, err := postgresConnection()
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }

        conn, err := pgx.Connect(r.Context(), connectionString)
        if err != nil {
            http.Error(w, "connect to PostgreSQL", http.StatusBadGateway)
            return
        }
        defer conn.Close(r.Context())

        counter := postgresCounter{conn: conn}
        switch r.URL.Path {
        case "/batch":
            createBatch(w, r, counter)
        case "/callback":
            completeItem(w, r, counter)
        default:
            http.NotFound(w, r)
        }
    }

    func createBatch(w http.ResponseWriter, r *http.Request, counter postgresCounter) {
        var input struct {
            Size int `json:"size"`
        }
        if err := json.NewDecoder(r.Body).Decode(&input); err != nil ||
            input.Size < 1 {
            http.Error(w, "expected a positive size", http.StatusBadRequest)
            return
        }

        var batchID string
        err := counter.conn.QueryRow(r.Context(), `
            INSERT INTO batch_counters (batch_id, remaining)
            VALUES (gen_random_uuid()::text, $1)
            RETURNING batch_id`, input.Size).Scan(&batchID)
        if err != nil {
            http.Error(w, "create batch", http.StatusBadGateway)
            return
        }

        writeJSON(w, http.StatusCreated, map[string]any{
            "batch_id": batchID,
            "remaining": input.Size,
        })
    }

    func completeItem(w http.ResponseWriter, r *http.Request, counter postgresCounter) {
        batchID := strings.TrimSpace(r.URL.Query().Get("batch_id"))
        if batchID == "" {
            http.Error(w, "batch_id is required", http.StatusBadRequest)
            return
        }

        var remaining int
        var decremented bool
        err := counter.conn.QueryRow(r.Context(), decrementQuery, batchID).
            Scan(&remaining, &decremented)
        if errors.Is(err, pgx.ErrNoRows) {
            http.Error(w, "batch not found", http.StatusNotFound)
            return
        }
        if err != nil {
            http.Error(w, "update batch", http.StatusBadGateway)
            return
        }

        complete := remaining == 0
        if complete && decremented {
            if err := notify(r.Context(), batchID); err != nil {
                http.Error(w, err.Error(), http.StatusBadGateway)
                return
            }
        }

        writeJSON(w, http.StatusOK, map[string]any{
            "batch_id": batchID,
            "remaining": remaining,
            "complete": complete,
        })
    }

    func postgresConnection() (string, error) {
        value, err := os.ReadFile(connectionSecret)
        if err != nil {
            return "", err
        }
        return strings.TrimSpace(string(value)), nil
    }

    func notify(parent context.Context, batchID string) error {
        body, _ := json.Marshal(map[string]string{
            "batch_id": batchID,
            "status": "complete",
        })

        ctx, cancel := context.WithTimeout(parent, notifyTimeout)
        defer cancel()

        req, err := http.NewRequestWithContext(
            ctx, http.MethodPost, os.Getenv("notify_url"), bytes.NewReader(body),
        )
        if err != nil {
            return err
        }
        req.Header.Set("Content-Type", "application/json")

        res, err := http.DefaultClient.Do(req)
        if err != nil {
            return err
        }
        defer res.Body.Close()

        if res.StatusCode < 200 || res.StatusCode >= 300 {
            return fmt.Errorf("notification endpoint returned %s", res.Status)
        }
        return nil
    }

    func writeJSON(w http.ResponseWriter, status int, value any) {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(status)
        json.NewEncoder(w).Encode(value)
    }
    ```

    Add `github.com/jackc/pgx/v5` to `fan-in/go.mod`.

=== "Python"

    `fan-in/handler.py`:

    ```python
    import json
    import os

    import psycopg
    import requests


    CONNECTION_SECRET = "/var/openfaas/secrets/postgres-connection"
    NOTIFY_TIMEOUT = 5

    DECREMENT_QUERY = """
    WITH updated AS (
        UPDATE batch_counters
           SET remaining = remaining - 1
         WHERE batch_id = %s
           AND remaining > 0
     RETURNING remaining
    )
    SELECT remaining, true
      FROM updated
    UNION ALL
    SELECT remaining, false
      FROM batch_counters
     WHERE batch_id = %s
       AND NOT EXISTS (SELECT 1 FROM updated)
    LIMIT 1
    """


    def handle(event, context):
        try:
            with psycopg.connect(postgres_connection(), autocommit=True) as connection:
                if event.path == "/batch":
                    return create_batch(event, connection)
                if event.path == "/callback":
                    return complete_item(event, connection)
                return error(404, "not found")
        except (KeyError, OSError, psycopg.Error, requests.RequestException) as err:
            return error(502, str(err))


    def create_batch(event, connection):
        try:
            size = int(json.loads(event.body)["size"])
        except (KeyError, TypeError, ValueError):
            return error(400, "expected a positive size")

        if size < 1:
            return error(400, "expected a positive size")

        row = connection.execute(
            """
            INSERT INTO batch_counters (batch_id, remaining)
            VALUES (gen_random_uuid()::text, %s)
            RETURNING batch_id
            """,
            (size,),
        ).fetchone()

        return {
            "statusCode": 201,
            "body": {"batch_id": row[0], "remaining": size},
        }


    def complete_item(event, connection):
        batch_id = event.query.get("batch_id", "").strip()
        if not batch_id:
            return error(400, "batch_id is required")

        row = connection.execute(
            DECREMENT_QUERY,
            (batch_id, batch_id),
        ).fetchone()
        if row is None:
            return error(404, "batch not found")

        remaining, decremented = row
        complete = remaining == 0
        if complete and decremented:
            response = requests.post(
                os.environ["notify_url"],
                json={"batch_id": batch_id, "status": "complete"},
                timeout=NOTIFY_TIMEOUT,
            )
            response.raise_for_status()

        return {
            "statusCode": 200,
            "body": {
                "batch_id": batch_id,
                "remaining": remaining,
                "complete": complete,
            },
        }


    def postgres_connection():
        with open(CONNECTION_SECRET, encoding="utf-8") as secret:
            return secret.read().strip()


    def error(status_code, message):
        return {"statusCode": status_code, "body": message}
    ```

    Add `psycopg[binary]` and `requests` to `fan-in/requirements.txt`.

=== "Node.js"

    `fan-in/handler.js`:

    ```javascript
    'use strict'

    const fs = require('node:fs')
    const { Client } = require('pg')

    const connectionSecret = '/var/openfaas/secrets/postgres-connection'
    const notifyTimeout = 5000

    const decrementQuery = `
    WITH updated AS (
        UPDATE batch_counters
           SET remaining = remaining - 1
         WHERE batch_id = $1
           AND remaining > 0
     RETURNING remaining
    )
    SELECT remaining, true AS decremented
      FROM updated
    UNION ALL
    SELECT remaining, false AS decremented
      FROM batch_counters
     WHERE batch_id = $1
       AND NOT EXISTS (SELECT 1 FROM updated)
    LIMIT 1`

    module.exports = async (event, context) => {
      if (event.path !== '/batch' && event.path !== '/callback') {
        return respond(context, 404, 'not found')
      }

      const client = new Client({ connectionString: postgresConnection() })
      try {
        await client.connect()
        if (event.path === '/batch') {
          return await createBatch(event, context, client)
        }
        return await completeItem(event, context, client)
      } catch (error) {
        return respond(context, 502, error.message)
      } finally {
        await client.end()
      }
    }

    async function createBatch (event, context, client) {
      let size
      try {
        size = Number(JSON.parse(requestBody(event.body)).size)
      } catch (error) {
        return respond(context, 400, 'expected a positive size')
      }

      if (!Number.isInteger(size) || size < 1) {
        return respond(context, 400, 'expected a positive size')
      }

      const result = await client.query(
        `INSERT INTO batch_counters (batch_id, remaining)
         VALUES (gen_random_uuid()::text, $1)
         RETURNING batch_id`,
        [size]
      )

      return respond(context, 201, {
        batch_id: result.rows[0].batch_id,
        remaining: size
      })
    }

    async function completeItem (event, context, client) {
      const batchID = String((event.query || {}).batch_id || '').trim()
      if (!batchID) {
        return respond(context, 400, 'batch_id is required')
      }

      const result = await client.query(decrementQuery, [batchID])
      if (result.rows.length === 0) {
        return respond(context, 404, 'batch not found')
      }

      const remaining = Number(result.rows[0].remaining)
      const complete = remaining === 0
      if (complete && result.rows[0].decremented) {
        const response = await fetch(process.env.notify_url, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ batch_id: batchID, status: 'complete' }),
          signal: AbortSignal.timeout(notifyTimeout)
        })
        if (!response.ok) {
          throw new Error(`notification endpoint returned ${response.status}`)
        }
      }

      return respond(context, 200, {
        batch_id: batchID,
        remaining,
        complete
      })
    }

    function postgresConnection () {
      return fs.readFileSync(connectionSecret, 'utf8').trim()
    }

    function requestBody (body) {
      if (Buffer.isBuffer(body)) {
        return body.toString()
      }
      return String(body || '')
    }

    function respond (context, status, body) {
      const contentType = typeof body === 'object'
        ? 'application/json'
        : 'text/plain; charset=utf-8'
      return context
        .status(status)
        .headers({ 'Content-Type': contentType })
        .succeed(body)
    }
    ```

    Add `pg` to `fan-in/package.json`.

## Configure the function

=== "Go"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      fan-in:
        lang: golang-middleware
        handler: ./fan-in
        image: ttl.sh/openfaas-examples/fan-in:latest
        environment:
          notify_url: http://gateway.openfaas:8080/async-function/printer
        secrets:
          - postgres-connection
    ```

=== "Python"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      fan-in:
        lang: python3-http
        handler: ./fan-in
        image: ttl.sh/openfaas-examples/python-fan-in:latest
        environment:
          notify_url: http://gateway.openfaas:8080/async-function/printer
        secrets:
          - postgres-connection
    ```

=== "Node.js"

    `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: http://127.0.0.1:8080
    functions:
      fan-in:
        lang: node24
        handler: ./fan-in
        image: ttl.sh/openfaas-examples/node-fan-in:latest
        environment:
          notify_url: http://gateway.openfaas:8080/async-function/printer
        secrets:
          - postgres-connection
    ```

The `notify_url` setting is use to configure the function or HTTP endpoint
invoked after the complete batch finishes.

## Deploy and try Fan-in

Deploy the `printer` function and follow its logs:

```bash
faas-cli store deploy printer
faas-cli logs printer -t
```

Deploy `fan-in` after completing the
[Fan-out example](/languages/patterns/fan-out/):

```bash
faas-cli up --tag=sha
```

Create a counter for the three-item batch and save its generated ID:

```bash
batch_id=$(curl -s http://127.0.0.1:8080/function/fan-in/batch \
  -H "Content-Type: application/json" \
  -d '{"size":3}' | jq -r .batch_id)
```

Submit the same batch from the Fan-out example, using `fan-in` as the callback:

```bash
curl -s http://127.0.0.1:8080/function/fan-out \
  -H "Content-Type: application/json" \
  -H "X-Callback-Url: http://gateway.openfaas:8080/function/fan-in/callback?batch_id=${batch_id}" \
  -d '["one", "two", "three"]'
```

Each callback decrements the counter. After the third callback, the `printer`
logs receive one completion message:

```json
{
  "batch_id": "4876cf09-a90e-40ba-9ac7-56fc87dd536b",
  "status": "complete"
}
```

## Fan-in considerations

* Progress must be stored in shared storage and updated atomically because
  callbacks may be concurrent and may reach different `fan-in` replicas.
* To combine results, persist each callback body with its batch ID and have the
  final function read them after the counter reaches zero. See the
  [original Fan-out and Fan-in example](https://www.openfaas.com/blog/fan-out-and-back-in-using-functions/)
  for a larger workflow that stores and combines individual results.
