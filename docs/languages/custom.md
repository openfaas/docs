## Custom Languages

An OpenFaaS function is a container image or Open Container Initiative (OCI) image. This means that you can use any programming language or toolchain that can be packaged into a container image.

### Source a template from the store

There are a number of officially supported templates, and community templates for other languages.

Anyone can write a template to support a new language or toolchain by following the example set out in an existing template.

View official and recommended templates:

```bash
faas-cli template store list --official --recommended
```

Find all official templates:

```bash
faas-cli template store list --official
```

List all templates including community templates, from the store:

```bash
faas-cli template store list
```

This will include code submitted by a third party. It may or may not be kept up to date or actively maintained, so you'll need to do your own due diligence.

### Make your own template

The easiest way to write a template for OpenFaaS it to copy an official template, and to modify it. This example creates a template using Bun and Express.js which is based upon the official 22 template.

Create a new folder called my-templates, it should have a sub-directory within it called `template`.

```bash
mkdir -p my-templates/template
```

For each template you wish to host, create a new folder i.e.

```bash
mkdir -p my-templates/template/bun-express
```

Create a template.yml file to specify the name and any welcome message:

```yaml
language: bun-express
welcome_message: |
  This template uses the Bun framework and Express.js to 
  build functions. To install packages run: bun install.
```

Next, create a Dockerfile which should download and make available the OpenFaaS watchdog.

There are two watchdogs available, the [Classic Watchdog](https://github.com/openfaas/classic-watchdog), meant for processes which do not have their own HTTP server, the process will be forked for each request, and the [of-watchdog](https://github.com/openfaas/of-watchdog) for when a HTTP server is available. Since Bun used with Express.js provides a HTTP server, we'll be using the of-watchdog. 

The following Dockerfile is based upon the node22 template, adapted for Bun.

The `--platform=${TARGETPLATFORM:-linux/amd64}` directive is required for multi-arch support, to make the template work for 64-bit Arm as well as regular *x86_64* machines. Most images that you find on the Docker Hub will already have multi-arch support, and it's strongly recommended to keep it in place.

```Dockerfile
FROM --platform=${TARGETPLATFORM:-linux/amd64} ghcr.io/openfaas/of-watchdog:0.10.9 as watchdog
FROM --platform=${TARGETPLATFORM:-linux/amd64} oven/bun:alpine as ship

ARG TARGETPLATFORM
ARG BUILDPLATFORM

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

RUN addgroup -S app && adduser app -S -G app

WORKDIR /root/

RUN mkdir -p /home/app

# Wrapper/boot-strapper
WORKDIR /home/app
COPY package.json ./
COPY jsconfig.json ./
COPY bun.lockb ./

# This ordering means the npm installation is cached for the outer function handler.
RUN bun install --production

# Copy outer function handler
COPY index.js ./

# COPY function node packages and install, adding this as a separate
# entry allows caching of npm install runtime dependencies
WORKDIR /home/app/function
COPY function/*.json ./
RUN bun install --production || :

# Copy in additional function files and folders
COPY --chown=app:app function/ .

WORKDIR /home/app/

# chmod for tmp is for a buildkit issue (@alexellis)
RUN chmod +rx -R ./function \
    && chown app:app -R /home/app \
    && chmod 777 /tmp

USER app

ENV cgi_headers="true"
ENV fprocess="bun run index.js"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:3000"

ENV prefix_logs="false"

HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]
```

You can find out more about [watchdog configuration options on GitHub](https://github.com/openfaas/of-watchdog/)

The function's index.js code should be placed in the `./my-templates/template/bun-express/` folder.

```js
// Copyright (c) Alex Ellis 2024. All rights reserved.
// Copyright (c) OpenFaaS Author(s) 2024. All rights reserved.
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

"use strict"

import express from "express";
const app = express()
const handler = require('./function/handler');
const bodyParser = require('body-parser')

const defaultMaxSize = '100kb' // body-parser default

app.disable('x-powered-by');

const rawLimit = process.env.MAX_RAW_SIZE || defaultMaxSize
const jsonLimit = process.env.MAX_JSON_SIZE || defaultMaxSize

app.use(function addDefaultContentType(req, res, next) {
    // When no content-type is given, the body element is set to 
    // nil, and has been a source of contention for new users.

    if(!req.headers['content-type']) {
        req.headers['content-type'] = "text/plain"
    }
    next()
})

if (process.env.RAW_BODY === 'true') {
    app.use(bodyParser.raw({ type: '*/*' , limit: rawLimit }))
} else {
    app.use(bodyParser.text({ type : "text/*" }));
    app.use(bodyParser.json({ limit: jsonLimit}));
    app.use(bodyParser.urlencoded({ extended: true }));
}

const isArray = (a) => {
    return (!!a) && (a.constructor === Array);
};

const isObject = (a) => {
    return (!!a) && (a.constructor === Object);
};

class FunctionEvent {
    constructor(req) {
        this.body = req.body;
        this.headers = req.headers;
        this.method = req.method;
        this.query = req.query;
        this.path = req.path;
    }
}

class FunctionContext {
    constructor(cb) {
        this.statusCode = 200;
        this.cb = cb;
        this.headerValues = {};
        this.cbCalled = 0;
    }

    status(statusCode) {
        if(!statusCode) {
            return this.statusCode;
        }

        this.statusCode = statusCode;
        return this;
    }

    headers(value) {
        if(!value) {
            return this.headerValues;
        }

        this.headerValues = value;
        return this;    
    }

    succeed(value) {
        let err;
        this.cbCalled++;
        this.cb(err, value);
    }

    fail(value) {
        let message;
        if(this.status() == "200") {
            this.status(500)
        }

        this.cbCalled++;
        this.cb(value, message);
    }
}

const middleware = async (req, res) => {
    const cb = (err, functionResult) => {
        if (err) {
            console.error(err);

            return res.status(fnContext.status())
                .send(err.toString ? err.toString() : err);
        }

        if(isArray(functionResult) || isObject(functionResult)) {
            res.set(fnContext.headers())
                .status(fnContext.status()).send(JSON.stringify(functionResult));
        } else {
            res.set(fnContext.headers())
                .status(fnContext.status())
                .send(functionResult);
        }
    };

    const fnEvent = new FunctionEvent(req);
    const fnContext = new FunctionContext(cb);

    Promise.resolve(handler(fnEvent, fnContext, cb))
    .then(res => {
        if(!fnContext.cbCalled) {
            fnContext.succeed(res);
        }
    })
    .catch(e => {
        cb(e);
    });
};

app.use(middleware)

const port = process.env.http_port || 3000;

app.listen(port, () => {
    console.log(`bun-express listening on port: ${port}`)
});
```

It starts a HTTP server on port 3000, then forwards all requests to the function handler, including abstracting the HTTP request into an event and context object.

The function handler is then called with the event and context objects, and a callback function.

Here's the code for `./my-templates/template/bun-express/function/handler.js`:

```js
'use strict'

module.exports = async (event, context) => {
  const result = {
    'body': JSON.stringify(event.body),
    'content-type': event.headers["content-type"]
  }

  return context
    .status(200)
    .succeed(result)
}
```

The code for the handler is always placed in a folder called `function`, and then the index.js and Dockerfile are kept hidden from the user.

For a C# template, there'd be a Dockerfile and Program.cs kept at the root level, hidden away from the user, with a sub-folder called `function` containing the function's code, this is a pattern you'll see in all of the official templates.

To test out your template we'd recommend saving `my-templates` as a Git repository, and then running `faas-cli template pull` along with the path to your repository. For example:

```
faas-cli template pull https://github.com/alexellis/my-templates
```

You can run the `faas-cli new --list` to view the templates available in the current directory:

```
Languages available as templates:
- bun-express
```

As an alternative to using `faas-cli template pull` and a remote Git repository, you can do local testing by copying the `template` folder into the same directory where you want to run `faas-cli new`.

Finally, you can try out the function with `local-run` or `faas-cli up`

```bash
export OPENFAAS_PREFIX=ttl.sh/test

faas-cli new --lang bun-express bun-fn

faas-cli local-run -f bun-fn.yml

curl -d "hello" http://localhost:8080
```

### Deploy an existing microservices or container image

To deploy an existing microservice or container image, it'll need to be listening to HTTP traffic on port 8080.

See also: [Workloads](/reference/workloads)
