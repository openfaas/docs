## Python

For more on Python, see the *python-http* and *python3-http-debian* templates under the [*Create new functions* page](/cli/templates).

There are two recommended templates for Python 3 users.

* python3-http - based upon Alpine Linux, small image size, for pure Python only.
* python3-http-debian - based upon Debian Linux, larger image size, required for native C modules as as SQL, Kafka, Pandas, and image manipulation.

[Flask](https://flask.palletsprojects.com/en/3.0.x/) is used internally for handling HTTP requests and responses, however it is not exposed to the user, so is only an implementation detail.

The HTTP server used is currently [Waitress](https://docs.pylonsproject.org/projects/waitress/en/latest/).

> This is an official template maintained by OpenFaaS Ltd.

## Downloading the templates

Using template pull with the repository's URL:

```bash
faas-cli template pull https://github.com/openfaas/python-flask-template
```

Using the template store, since they are both within the same repository, either command will fetch down both templates.

```bash

faas-cli template store pull python3-http
faas-cli template store pull python3-http-debian
```

Using your `stack.yml` file:

```yaml
configuration:
    templates:
        - name: python3-http
```

### Python 3 versions

The python3-http template supports a dynamic Python version through a build argument.

It is recommended that you put the version within the OpenFaaS YAML file:

```diff
functions:
  postgres-fn:
    lang: python3-http
+    build_args:
+      - PYTHON_VERSION=3.12
```

The option can also be passed via a CLI flag, however this is not recommended as it is easy to forget or miss it:

```bash
faas-cli build --build-arg PYTHON_VERSION=3.11
```

## Event and Context Data

The function handler is passed two arguments, *event* and *context*.

*event* contains data about the request, including:
- body
- headers
- method
- query
- path

*context* contains basic information about the function, including:
- hostname

## Response Bodies

By default, the template will automatically attempt to set the correct Content-Type header for you based on the type of response.

For example, returning a dict object type will automatically attach the header `Content-Type: application/json` and returning a string type will automatically attach the `Content-Type: text/html, charset=utf-8` for you.

### Accessing Event Data

Accessing request body:
```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": "You said: " + str(event.body)
    }
```
Accessing request method:

```python
def handle(event, context):
    if event.method == 'GET':
        return {
            "statusCode": 200,
            "body": "GET request"
        }
    else:
        return {
            "statusCode": 405,
            "body": "Method not allowed"
        }
```
Accessing request query string arguments:

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": {
            "name": event.query['name']
        }
    }
```
Accessing request headers:

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": {
            "content-type-received": event.headers.get('Content-Type')
        }
    }
```

### Return a JSON body with a Content-Type

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": {"message": "Hello from OpenFaaS!"},
        "headers": {
            "Content-Type": "application/json"
        }
    }
```

### Custom status codes and response bodies

Successful response status code and JSON response body

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": {
            "key": "value"
        }
    }
```

Successful response status code and string response body

```python
def handle(event, context):
    return {
        "statusCode": 201,
        "body": "Object successfully created"
    }
```

Failure response status code and JSON error message

```python
def handle(event, context):
    return {
        "statusCode": 400,
        "body": {
            "error": "Bad request"
        }
    }
```

### Custom Response Headers

Setting custom response headers

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": {
            "key": "value"
        },
        "headers": {
            "Location": "https://www.example.com/"
        }
    }
```

### Native dependencies

As explained in the introduction, only the Debian variant of the Python template is suitable for building native dependencies. Why? For one, many libraries are available as pre-compiled wheels, meaning they can be imported without any compilation. Secondly, Alpine Linux requires so many packages to be added to build code that it becomes larger than the Debian base. Thirdly, Alpine Linux is not compatible with many native libraries because it uses its own C library called musl.

If a pre-compiled wheel isn't available for your chosen package, then you can use a build option to add a build toolchain. Build options are an abstracted list of packages to install, grouped together.

```diff
functions:
  postgres-fn:
    lang: python3-http
+    build_options:
+      - libpq
```

The current list of *build_options* for the Debian-based template is available in the templates repository in the [template.yml file](https://github.com/openfaas/python-flask-template/blob/master/template/python3-http-debian/template.yml). Pull requests and contributions are welcome, however packages can be specified even when they are not present as a build option.

Alternatively, individual packages within apt can be specified through build_args:

```diff
functions:
  postgres-fn:
    lang: python3-http
+    build_args:
+      ADDITIONAL_PACKAGE: "libpq-dev gcc python3-dev"
```

### Example with Postgresql

stack.yml

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  pgfn:
    lang: python3-http-debian
    handler: ./pgfn
    image: pgfn:latest
    build_options:
      - libpq
```

Alternatively you can specify `ADDITIONAL_PACKAGE` in the `build_args` section for the function.

```yaml
    build_args:
      ADDITIONAL_PACKAGE: "libpq-dev gcc python3-dev"
```

requirements.txt

```
psycopg2==2.9.3
```

Create a database and table:

```sql
CREATE DATABASE main;

\c main;

CREATE TABLE users (
    name TEXT,
);

-- Insert the original Postgresql author's name into the test table:

INSERT INTO users (name) VALUES ('Michael Stonebraker');
```

handler.py:

```python
import psycopg2

def handle(event, context):

    try:
        conn = psycopg2.connect("dbname='main' user='postgres' port=5432 host='192.168.1.35' password='passwd'")
    except Exception as e:
        print("DB error {}".format(e))
        return {
            "statusCode": 500,
            "body": e
        }

    cur = conn.cursor()
    cur.execute("""SELECT * from users;""")
    rows = cur.fetchall()

    return {
        "statusCode": 200,
        "body": rows
    }
```

Always read the secret from an OpenFaaS secret at `/var/openfaas/secrets/secret-name`. The use of environment variables is an anti-pattern and will be visible via the OpenFaaS API.

### Authenticate a function

To authenticate a function with a pre-shared secret, or API token, first create a secret, bind that secret to the function, then read it at runtime and validate it.

Create a new pre-shared secret:

```bash
openssl rand -base64 32 > python-auth-token.txt
```

Then create a new secret in OpenFaaS:

```bash
faas-cli secret create python-auth-token \
    --from-file python-auth-token.txt
```

Create a new function called `python-auth`:

```bash
faas-cli new --lang python3-http \
    python-auth
```

Bind the secret to the function:

```diff
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  python-auth:
    lang: python3-http-debian
+    secrets:
+    - python-auth-token
```

Now edit the function's `handler.py` file to read the secret and validate it:

```python
def read_secret(name):
    with open("/var/openfaas/secrets/" + name) as f:
        return f.read().strip()

def handle(event, context):
    token = read_secret("python-auth-token")

    if not "Authorization" in event.headers:
        return {
            "statusCode": 401,
            "body": "Unauthorized"
        }

    if event.headers["Authorization"] != "Bearer {}".format(token):
        return {
            "statusCode": 401,
            "body": "Unauthorized"
        }

    return {
        "statusCode": 200,
        "body": "Access granted"
    }
```

Deploy the function with `faas-cli up`, then invoke it:

```bash
curl -i https://127.0.0.1:8080/function/python-auth \
    -H "Authorization: Bearer $(cat ./python-auth-token.txt)"

HTTP/2 200
Access granted
```

