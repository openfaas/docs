## Node.js

The Node.js template for OpenFaaS uses Express.js under the hood, but provides an abstraction where you just work with an event and context object.

The event is used to obtain the original HTTP request, and the context is used to set the HTTP response. The underlying Express.js object is an implementation detail, and so is not available to the function author.

Async/await is supported by the handler by default.

The most thorough and complete examples for JavaScript/Node for OpenFaaS are in Alex Ellis' eBook: [Serverless for Everyone Else](https://openfaas.gumroad.com/l/serverless-for-everyone-else)

### Create a new function

Create a new function using the template:

```bash
faas-cli template pull
faas-cli new --lang node18 echo
```

You'll find a new folder called `echo` with a `handler.js` and `package.json` file inside.

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

### The Event object

The event object has the following properties:

* `event.body` - the body of the HTTP request, either as a string or a JSON object
* `event.headers` - the HTTP headers as a JSON object, index them as a dictionary i.e. `event.headers["content-type"]`
* `event.method` - the HTTP method as a string
* `event.query` - the query string as a JSON object, index them as a dictionary i.e. `event.query
* `event.path` - the path of the HTTP request

### The Context object

The context object has the following methods:

* `context.status(code)` - set the HTTP status code
* `context.succeed(result)` - set the HTTP response body, and end the request
* `context.fail(error)` - set the HTTP status code to 500, and end the request
* `context.headers(headers)` - set the HTTP headers, pass in a JSON object

You may wish to combine the `context.headers()` method with the `context.succeed()` method to set the `content-type` header.

Or to combine the `context.status()` method with the `context.succeed()` method to set an alternative successful HTTP status code such as a 201.

### Install packages

To install packages, `cd` into the function folder and run `npm install --save`:

```bash
faas-cli new --lang node18 http-req
cd http-req

npm install --save axios
```

Edit the `handler.js`:

```js
'use strict'

const axios = require('axios')

module.exports = async (event, context) => {
  if(!event.body.url) {
    return context
      .status(400)
      .fail('Missing .url in request body')
  }

  var result = {}

  try {
    const res = await axios({"method": "GET", "url": event.body.url, validateStatus: () => true})
    result["body"] = res.data
    result["status"] = res.status
  } catch (e){
    return context
      .status(500)
      .fail(e)
  }

  return context
    .status(result.status)
    .succeed(result.body)
}
```

Run the function with `faas-cli local-run`, and then test it out with a valid JSON input:

```bash
curl -i http://127.0.0.1:8080 \
    -H "Content-type: application/json" \
    --data '{"url":"https://openfaas.com"}'
```

As you can see from the example, when a valid JSON input is used in the request, along with an appropriate "Content-type" header, the event.body will transform into an object, which can be indexed as a dictionary.

### Unit tests

Unit tests provide a quick and efficient way to exercise your code on your local computer, without needing to run `faas-cli build` or to deploy the function to a remote cluster.

With the node18 template, any unit tests that you provide will be run automatically upon each invocation of `faas-cli build`.


By default, an empty test step is written to package.json inside your function's handler folder, you can override this with your own command or test runner.

For example:

```json
{
  "name": "function",
  "version": "1.0.0",
  "description": "",
  "main": "handler.js",
  "scripts": {
    "test": "mocha test/test.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "chai": "^4.2.0",
    "mocha": "^7.0.1"
  }
}
```

Then create at least one test file such as: `function-name/test/test.js`:

```js
var chai = require("chai")
var expect = chai.expect;

describe('MyFunction', function() {
  expect("foobar").to.have.lengthOf(3);
})
```

If the tests fail, this will also fail the build of your function and prevent it from passing.

For a more detailed example, see: [Serverless for Everyone Else](https://gumroad.com/l/serverless-for-everyone-else)