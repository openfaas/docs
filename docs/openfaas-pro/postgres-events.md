# Trigger functions from Postgres

Trigger functions from [PostgreSQL](https://www.postgresql.org/) tables efficiently without the overhead of database triggers.

> Note: This feature is included for [OpenFaaS Pro](https://openfaas.com/support/) customers.

## Installation

There are two options for triggering functions from Postgres:

1) Using logical replication and the Write Ahead Log (WAL) (explained here)
2) Using LISTEN/NOTIFY and a series of table-level triggers (contact us for instructions)

### Configure your Postgresql database

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

### Install the connector with Helm

* Set up the connector

    You can install the SQS connector using its [helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/postgres-connector).

    The values.yaml file can be customised to suit your needs.

    Provide the connection string in a Kubernetes secret as directed.

* Tuning the connector for your needs

    `filters` - set this to filter which tables and events to be notified about i.e. `customers:insert,customers:delete,subscriptions:update`

## Usage

You can deploy our [tester function](https://github.com/openfaas/store-functions/blob/master/printer/handler.go) which prints out the HTTP headers and body of any invocation.

A function can subscribe to a subset of the values you set in the `filters` field in values.yaml when installing with helm.

Separate each one with a comma, as follows, or give just one if that's what you need.

```bash
faas-cli deploy \
    --name printer \
    --image ghcr.io/openfaas/printer:latest \
    --annotation topic=customer:insert,customer:update
```

Create a test table and insert some data:

```bash
CREATE TABLE customer (id integer primary key generated always as identity, name text, created_at timestamp);

insert into customer (name, created_at) values ('Alex Ellis', now());
```

You should see the following output:

```bash
faas-cli logs printer

2022-12-05T17:06:30Z X-Forwarded-For=[127.0.0.1:38194]
2022-12-05T17:06:30Z X-Forwarded-Host=[127.0.0.1:8080]
2022-12-05T17:06:30Z X-Start-Time=[1670259990419597239]
2022-12-05T17:06:30Z Content-Type=[application/json]
2022-12-05T17:06:30Z X-Call-Id=[8df2faeb-acff-496d-8a95-1edbd18168dd]
2022-12-05T17:06:30Z X-Event-Action=[delete]
2022-12-05T17:06:30Z User-Agent=[Go-http-client/1.1]
2022-12-05T17:06:30Z X-Event-Table=[customer]
2022-12-05T17:06:30Z Accept-Encoding=[gzip]
2022-12-05T17:06:30Z X-Topic=[customer:delete]
2022-12-05T17:06:30Z X-Message-Id=[7]
2022-12-05T17:06:30Z X-Connector=[connector-sdk openfaasltd/postgres-connector]
2022-12-05T17:06:30Z X-Event-Id=[0ae31f36-71a7-498b-af11-f3b67d4422e2]
2022-12-05T17:06:30Z 
2022-12-05T17:06:30Z {"id":"0ae31f36-71a7-498b-af11-f3b67d4422e2","schema":"public","table":"customer","action":"delete","data":{"id":44,"name":null,"created_at":null},"commitTime":"2022-12-05T17:07:37.802252Z"}
2022-12-05T17:06:30Z 
2022-12-05T17:06:30Z 2022/12/05 17:06:30 POST / - 202 Accepted - ContentLength: 0B (0.0007s)
```

## Reference

The content-type is set to `application/json` and the body is a JSON object.

The body will contain JSON as follows for an insert:

```json
{
  "id": "3c8aabd7-0229-4c76-b05c-552ac61831ad",
  "schema": "public",
  "table": "customer",
  "action": "insert",
  "data": {
    "id": 44,
    "name": "Alex Ellis",
    "created_at": "2022-12-05 17:07:33.245422"
  },
  "commitTime": "2022-12-05T17:07:33.246235Z"
}
```

For a delete:

```json
{
  "id": "0ae31f36-71a7-498b-af11-f3b67d4422e2",
  "schema": "public",
  "table": "customer",
  "action": "delete",
  "data": {
    "id": 44,
    "name": null,
    "created_at": null
  },
  "commitTime": "2022-12-05T17:07:37.802252Z"
}
```

Additional headers are also made available, which mean you can efficiently filter out events that you do not need to process, without parsing JSON.

* `X-Event-Action` - the action that was performed on the table, e.g. `insert`, `update`, `delete`
* `X-Event-Table` - the table that was affected
* `X-Topic` - the topic that was used to subscribe to the event i.e. `customer:insert`
* `X-Event-Id` - a UUID for the delivery of this event 
* `X-Message-Id` - the message ID of the event - increments from 0 to N based upon the amount of events received by the connector

## Would you like a demo?

Feel free to reach out to us for a demo or to ask any questions you may have.

* [Let's talk](https://openfaas.com/support/)
