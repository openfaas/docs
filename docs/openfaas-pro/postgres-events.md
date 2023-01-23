# Trigger functions from Postgres

Trigger functions from [PostgreSQL](https://www.postgresql.org/) tables efficiently without the overhead of database triggers.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

## Installation

There are two options for triggering functions from Postgres:

1. Using logical replication and the Write Ahead Log (WAL)
2. Using LISTEN/NOTIFY and a series of table-level triggers

### Configure your Postgresql database

=== "WAL mode"

    You can configure a cloud/managed Postgresql database or install Postgres locally, the following settings are required in `postgresql.conf`:

    ```yaml
    wal_level = logical
    max_replication_slots = 5
    max_wal_senders = 10
    ```

    You can set up a self-hosted postgres database to test out this feature:

    ```
    docker run --rm --name pg -e POSTGRES_PASSWORD=passwd -p 5432:5432 -ti postgres:latest

    docker cp pg:/var/lib/postgresql/data/postgresql.conf postgresql.conf

    echo "wal_level = logical" | tee -a postgresql.conf
    echo "max_wal_senders = 10" | tee -a postgresql.conf
    echo "max_replication_slots = 5" | tee -a postgresql.conf
    docker cp postgresql.conf pg:/var/lib/postgresql/data/postgresql.conf
    docker restart pg
    ```

    Connect and test it:

    ```bash
    PGPASSWORD=passwd psql -U postgres -h 127.0.0.1
    ```

=== "Trigger mode"

    The connector will automatically create a database schema named `openfaas` and a function `openfaas.notify_event` when trigger mode is enabled.
    
    To receive events on table changes a trigger needs to be created for one or more specific tables to. The [usage](#usage) section explains how to create these.

### Install the connector with Helm

* Set up the connector

    You can install the Postgres connector using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/postgres-connector).

    The `values.yaml` file can be customised to suit your needs.

    Provide the connection string in a Kubernetes secret as directed.

* Tuning the connector for your needs.

    `filters` - set this to filter which tables and events to be notified about i.e. `customers:insert,customers:delete,subscriptions:update`

## Usage

You can deploy our [tester function](https://github.com/openfaas/store-functions/blob/master/printer/handler.go) which prints out the HTTP headers and body of any invocation.

A function can subscribe to a subset of the values you set in the `filters` field in values.yaml when installing with helm.

Separate each one with a comma, as follows, or give just one if that's what you need.

```bash
faas-cli store deploy printer \
    --annotation topic=customer:insert,customer:update
```

### Create a table and insert data.

=== "WAL mode"

    1. Create a table i.e. `customer`:
      ```sql
      CREATE TABLE customer (
        id integer primary key generated always as identity,
        name text,
        created_at timestamp);
      ```
    2. Insert some data into the table:
      ```sql
      insert into customer (name, created_at) values ('Alex Ellis', now());
      ```

=== "Trigger mode"

    1. Create a table i.e. `customer`:
      ```sql
      CREATE TABLE customer (
        id integer primary key generated always as identity,
        name text,
        created_at timestamp);
      ```
    2. Create a trigger for the `customer` table:
      ```sql
      CREATE TRIGGER customer_notify_event
      AFTER INSERT OR UPDATE OR DELETE ON customer
          FOR EACH ROW EXECUTE PROCEDURE openfaas.notify_event();
      ```
    3. Insert some data into the table:
      ```sql
      insert into customer (name, created_at) values ('Alex Ellis', now());
      ```


Take a look at the logs of the printer function to inspect the invocations made by the connector. You should see the following output:

=== "WAL mode"

    ```bash
    faas-cli logs printer

    2023-01-23T17:13:50Z X-Event-Id=[e6b7da06-775a-4220-becb-f483508e1eb9]
    2023-01-23T17:13:50Z X-Message-Id=[0]
    2023-01-23T17:13:50Z X-Connector=[connector-sdk openfaasltd/postgres-connector]
    2023-01-23T17:13:50Z X-Forwarded-Uri=[/function/printer.openfaas-fn]
    2023-01-23T17:13:50Z X-Start-Time=[1674494030262922184]
    2023-01-23T17:13:50Z Accept-Encoding=[gzip]
    2023-01-23T17:13:50Z X-Event-Action=[insert]
    2023-01-23T17:13:50Z X-Event-Table=[customer]
    2023-01-23T17:13:50Z Content-Type=[application/json]
    2023-01-23T17:13:50Z X-Call-Id=[9a7ad051-1038-4a04-bd16-e1a9d478855f]
    2023-01-23T17:13:50Z X-Topic=[customer:insert]
    2023-01-23T17:13:50Z 2023/01/23 17:13:50 POST / - 202 Accepted - ContentLength: 0B (0.0007s)
    2023-01-23T17:13:50Z User-Agent=[Go-http-client/1.1]
    2023-01-23T17:13:50Z 
    2023-01-23T17:13:50Z {"id":"e6b7da06-775a-4220-becb-f483508e1eb9","schema":"public","table":"customer","action":"insert","data":{"created_at":"2023-01-23T17:13:50.24367Z","id":2,"name":"Alex Ellis"},"dataOld":{},"commitTime":"2023-01-23T17:13:50.243871Z"}
    ```

=== "Trigger mode"
    ```bash
    faas-cli logs printer

    2023-01-23T17:32:31Z Accept-Encoding=[gzip]
    2023-01-23T17:32:31Z X-Event-Action=[insert]
    2023-01-23T17:32:31Z X-Call-Id=[93050c37-faa6-4e69-bc00-c0e15f17bea0]
    2023-01-23T17:32:31Z X-Forwarded-Uri=[/function/printer.openfaas-fn]
    2023-01-23T17:32:31Z X-Start-Time=[1674495151266155144]
    2023-01-23T17:32:31Z User-Agent=[Go-http-client/1.1]
    2023-01-23T17:32:31Z X-Event-Table=[customer]
    2023-01-23T17:32:31Z X-Topic=[customer:insert]
    2023-01-23T17:32:31Z X-Message-Id=[1]
    2023-01-23T17:32:31Z Content-Type=[application/json]
    2023-01-23T17:32:31Z X-Connector=[connector-sdk openfaasltd/postgres-connector]
    2023-01-23T17:32:31Z 2023/01/23 17:32:31 POST / - 202 Accepted - ContentLength: 0B (0.0006s)
    2023-01-23T17:32:31Z 
    2023-01-23T17:32:31Z {"schema":"public","table":"customer","action":"insert","data":{"created_at":"2023-01-23T17:32:31.248531","id":2,"name":"Alex Ellis"}}
    ```

## Reference

The content-type is set to `application/json` and the body is a JSON object.

The body will contain JSON as follows for an insert:

=== "WAL mode"

    ```json
    {
      "id": "e6b7da06-775a-4220-becb-f483508e1eb9",
      "schema": "public",
      "table": "customer",
      "action": "insert",
      "data": {
        "created_at": "2023-01-23T17:13:50.24367Z",
        "id": 2,
        "name": "Alex Ellis"
      },
      "dataOld": {},
      "commitTime": "2023-01-23T17:13:50.243871Z"
    }
    ```

=== "Trigger mode"

    ```json
    {
      "schema": "public",
      "table": "customer",
      "action": "insert",
      "data":{
        "created_at": "2023-01-23T17:32:31.248531",
        "id": 2,
        "name": "Alex Ellis"
      }
    }
    ```

For a delete:

=== "WAL mode"

    ```json
    {
      "id": "da83108d-9421-443e-a4ca-d8516598f82d",
      "schema": "public",
      "table": "customer",
      "action": "delete",
      "data": {},
      "dataOld": {
        "created_at": null,
        "id": 2,
        "name": null
      },
      "commitTime": "2023-01-23T17:28:20.552425Z"}
    ```

=== "Trigger mode"

    ```json
    {
      "schema": "public",
      "table": "customer",
      "action": "delete",
      "data": {
        "created_at": "2023-01-23T17:32:31.248531",
        "id": 2,
        "name": "Alex Ellis"
      }
    }
    ```

### Additional headers

Additional headers are also made available, which mean you can efficiently filter out events that you do not need to process, without parsing JSON.

=== "WAL mode"

    * `X-Event-Action` - the action that was performed on the table, e.g. `insert`, `update`, `delete`
    * `X-Event-Table` - the table that was affected
    * `X-Topic` - the topic that was used to subscribe to the event i.e. `customer:insert`
    * `X-Event-Id` - a UUID for the delivery of this event 
    * `X-Message-Id` - the message ID of the event - increments from 0 to N based upon the amount of events received by the connector

=== "Trigger mode"

    * `X-Event-Action` - the action that was performed on the table, e.g. `insert`, `update`, `delete`
    * `X-Event-Table` - the table that was affected
    * `X-Topic` - the topic that was used to subscribe to the event i.e. `customer:insert`


## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
