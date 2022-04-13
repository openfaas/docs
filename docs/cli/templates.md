# Create new functions

## Get started

Once you've installed the `faas-cli` you can start creating and deploying functions via the `faas-cli up` command or using the individual commands:

* `faas-cli build` - build an image into the local Docker library
* `faas-cli push` - push that image to a remote container registry
* `faas-cli deploy` - deploy your function into a cluster

The `faas-cli up` command automates all of the above in a single command.

For Raspberry Pi and ARM, you must use the `publish` command instead of `build` and `push`, or `up`.

See the notes here: [Building multi-arch images for ARM and Raspberry Pi](/cli/build/)

## Templates

The OpenFaaS CLI has a template engine built-in which can create new functions in a given programming language. The way this works is by reading a list of templates from the `./template` location in your current working folder.

Before creating a new function make sure you pull in the official OpenFaaS language templates from GitHub via the [templates repository](https://github.com/openfaas/templates).

```bash
$ faas-cli template pull
```

This page shows how to generate functions in the most popular languages and explains how you can manage their dependencies too.

### Classic vs. of-watchdog templates

The *Classic Templates* are held in the [openfaas/templates](https://github.com/openfaas/templates) repository and are based upon the *Classic Watchdog* which uses STDIO to communicate with your function. The of-watchdog uses HTTP to communicate with functions and most of its templates are available in the [openfaas-incubator](https://github.com/openfaas-incubator/) organisation on GitHub and in the store.

How to pick:

* Use the *Classic Watchdog* if you're starting out or following tutorials or guides
* Use the *of-watchdog* if you need more performance or if you need full control of the HTTP response

See also: [watchdog design](/architecture/watchdog/)

### Template store

You can browse templates from the official store or create your own store and add your own templates there.

To see what templates are available type `faas-cli template store list` and you should see the following in the terminal:

```sh
$ faas-cli template store list

NAME                    SOURCE             DESCRIPTION
csharp                  openfaas           Official C# template
dockerfile              openfaas           Official Dockerfile template
...
node10-express          openfaas-incubator NodeJS 10 Express template
ruby-http               openfaas-incubator Ruby 2.4 HTTP template
golang-middleware       openfaas Golang Middleware template
csharp-httprequest      distantcam         C# HTTP template
...
```

Choose a template and retrieve it locally with the command:

```sh
$ faas-cli template store pull node10-express
```

Once downloaded, your chosen template and any others stored in the same repository will be available to use:

```sh
$ faas-cli new --list
Languages available as templates:
- node10-express
```

You can add your own store just by specifying the `--url` flag for both commands to pull and list your custom templates store.

The classic templates are held in the [openfaas/templates](https://github.com/openfaas/templates) repository.

#### Go templates

There are several Golang templates available, which are listed below.

| Name                | Style        | Watchdog    | Dependencies            |
|:--------------------|:-------------|:------------|:------------------------|
| `go`                | Function     | classic     | vendoring or Go modules |
| `golang-middleware` | Microservice | of-watchdog | vendoring or Go modules |
| `golang-http`       | Function     | of-watchdog | vendoring or Go modules |

All templates are available via `faas-cli template store list/pull`

#### Go `golang-http` - (of-watchdog template)

[Read the README for golang-http](https://github.com/openfaas/golang-http-template), this template has a similar-style of API to AWS Lambda.

Golang modules are supported via `--build-arg` using `GO111MODULE=1` or `GO111MODULE=auto`

#### Go `golang-middleware` - (of-watchdog template)

[Read the README for golang-middleware](https://github.com/openfaas/golang-http-template), this template is ideal for full control over the HTTP request and response and corresponds to a HTTP middleware in Go.

```golang
func Handle(w http.ResponseWriter, r *http.Request) {
}
```

Golang modules are supported via `--build-arg` using `GO111MODULE=1` or `GO111MODULE=auto`

#### Go `go` - (classic template)

To create a new function named `go-fn` in Go type in the following:

```bash
$ faas-cli new go-fn --lang go
```

You will now see two files generate:

```sh
go-fn.yml
./go-fn/
./go-fn/handler.go
```

You can now edit `handler.go` and use the `faas-cli` to `build` and `deploy` your function.

##### Go `go` - dependencies

Dependencies should be managed with a Go modules. Vendoring is also supported when you need to use private code, see the next section.

You must set `GO111MODULE=on` in the `build_args` section of `stack.yml`, or pass the relevant flags with `faas-cli build/up/publish --build-arg key=value`.

```yaml
functions:
  with_go_modules:
    handler: ./with_go_modules
    lang: go
    build_args:
      GO111MODULE: on
```

Now use `go mod init function` to initialize your function. Once initialized, you can now use  `go get` to manage your dependencies.

###### Vendoring for use of private repos and modules

You must set `GO111MODULE=off` and `GOFLAGS=-mod=vendor` in the `build_args` section of `stack.yml`, or pass the relevant flags with `faas-cli build/up/publish --build-arg key=value`.

```yaml
    build_args:
      GO111MODULE: off
      GOFLAGS: "-mod=vendor"
```

###### Including sub-modules

If you would like to include sub-modules, a certain replace statement is required in your `go.mod` file: `replace handler/function => ./`. This replace statement allows Go to see and use all sub-modules you create with-in your handler, for example

Create your sub-package i.e. `handlers` and run `cd handlers ; go mod init`

Here's handlers/handlers.go:

```golang
package handlers

import (
	"fmt"

	execute "github.com/alexellis/go-execute/pkg/v1"
)

func Handle() {
	ls := execute.ExecTask{
		Command: "exit 1",
		Shell:   true,
	}
	res, err := ls.Execute()
	if err != nil {
		panic(err)
	}

	fmt.Printf("stdout: %q, stderr: %q, exit-code: %d\n", res.Stdout, res.Stderr, res.ExitCode)
}
```

Within your handler.go:

```golang
package function

import (
	"fmt"

	"handler/function/handlers"
)

// Handle a serverless request
func Handle(req []byte) string {

	handlers.Handle()

	return fmt.Sprintf("Hello, Go. You said: %s", string(req))
}
```

Now add the following replace statement to your `go.mod`

```
replace handler/function => ./
```

This can also be affected using

```sh
go mod edit -replace handler/function=./
```

Now you can build with `--build-arg GO111MODULE=on` or with a `build_arg` map entry for the function in its stack.yml.


###### With vendoring

You can also use vendoring with Go modules. This can reduce build times, help deal with private module dependencies, or help with ensuring reproducible builds.

* Now vendor a library

Go supports vendoring using `go mod vendor`. Make sure you're in the `go-fn` folder, now use `go mod vendor` to create (or later update) your vendor folder. Go will now use this vendor folder while building your function.


##### Go `go` - with CGO

First you will need to add the `dev` build option:

```yaml
    build_options:
    - dev
```

This installs `gcc`, `make`, `git` and some other related packages for the build portion of the function's Dockerfile.

You can then enable CGO with a build-arg:

```
faas-cli build --build-arg CGO_ENABLED=1
```

#### Python 3 templates

For production use, serving machine learning models, and high-traffic functions, it's advisable to use the newer templates built with flask and the OpenFaaS of-watchdog.

See the [python-flask-template repo](https://github.com/openfaas/python-flask-template) for the following templates:

* python27-flask
* python3-flask
* python3-flask-debian
* python3-http
* python3-http-debian

#### Python 3 (classic template)

To create a Python function named `pycon` type in:

```
$ faas-cli new pycon --lang python3
```

You'll see:

```
pycon.yml
pycon/handler.py
pycon/requirements.txt
```

> Note: Python 2.7 is also available with the language template `python`, but the Python community now consider [Python version 2.7 to be deprecated and end-of-life](https://www.python.org/dev/peps/pep-0373/).

##### Python: dependencies

You should edit `pycon/requirements.txt` and add any pip modules you want with each one on a new line, for instance `requests`.

The primary Python template uses Alpine Linux as a runtime environment due to its minimal size, but if you need a Debian environment so that you can compile `numpy` or other modules then read on to the next section.

##### Python: advanced dependencies

If you need to use pip modules that require compilation then you should try the python3-debian template then add your pip modules to the `requirements.txt` file.

```
$ faas-cli template pull
$ faas-cli new numpy-function --lang python3-debian
$ echo "numpy" > ./numpy-function/requirements.txt
$ faas-cli build -f ./numpy-function.yml
...

Step 11/17 : RUN pip install -r requirements.txt
 ---> Running in d0ff430a607e
Collecting numpy (from -r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/6e/dc/92c0f670e7b986829fc92c4c0208edb9d72908149da38ecda50d816ea057/numpy-1.14.2-cp36-cp36m-manylinux1_x86_64.whl (12.2MB)
Installing collected packages: numpy
Successfully installed numpy-1.14.2

...
```

#### Node.js templates (of-watchdog template)

There are three Node.js templates which use the newer of-watchdog:

| Name                   | Style         | Runtime | Version | async/await | Supported by nodejs.org |
|:-----------------------|:--------------|:--------|:--------|:------------|:------------------------|
| node12                 | Function      | NodeJS  | 12.x    | yes         | No        |
| node14                 | Function      | NodeJS  | 14.x    | yes         | yes, LTS version        |
| node16                 | Function      | NodeJS  | 16.x    | yes         | yes, "current" version        |


The `nodeXX` templates use event and context objects can be used to read the HTTP request and headers, and to set a response. You can learn about what's avaiable in the [index.js](https://github.com/openfaas/templates/blob/master/template/node12/index.js) file.

For a full reference guide to writing functions, managing secrets and connection pools see the manual: [Serverless for Everyone Else](https://gumroad.com/l/serverless-for-everyone-else)

##### Node.js templates - async/await

```js
"use strict"

module.exports = async (event, context) => {
    const result = {
        status: "Received input: " + JSON.stringify(event.body),
    };
    return result
}
```

##### Node.js templates - adding unit tests

Writing unit tests means that you get to exercise your code without building and deploying a container to OpenFaaS. It can save you a lot of time if you need to iterate often. It also means that you can provide tests that run every time your function is built.

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

##### Node.js templates - async/await with error

```js
"use strict"

module.exports = async (event, context) => {
    throw new Error("there was an error created in the function")
}
```

##### Node.js templates - without async/await

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

##### Node.js templates - Access to the raw body

Set the environment variable `RAW_BODY` to `true` to set the `context.body` to the original request body rather than the default behavior of parsing it as JSON.

This is useful where the original body needs to be passed to the function code without any parsing or processing. For instance, when working with binary data, or verifying the signature of a webhook.

```yaml
  environment:
    RAW_BODY: true
```

The raw body has a default maximum of 100KB to prevent abuse from users. This can be configured manually to deal with larger payloads:

```yaml
  environment:
    RAW_BODY: true
    MAX_RAW_BODY: 512kb
```

##### Node.js templates - Set max json request body size

Change the maximum size of a JSON request body by setting the environment variable `MAX_JSON_SIZE`. The default value is `'100kb'`
> Note: the value must be enclosed in quotes `'` `'`

This is useful when the function is expected to receive large amounts of JSON data in a single request. For instance, when working with large data sets and complex object types.

```yaml
  environment:
    MAX_JSON_SIZE: '5mb'
```

#### Node.js (classic template)

Generate a function named `js-fn`:

```bash
$ faas-cli new js-fn --lang node
```

You'll see:

```bash
./js-fn.yml
./js-fn/
./js-fn/handler.js
./js-fn/package.json
```

##### Node.js dependencies

Node.js dependencies are managed with `npm` and the `package.json` file which was generated for you.

To add the `cheerio` module type in:

```
cd js-fn
npm i --save cheerio
```

You can now add a `require('cheerio')` statement into your function and make use of this library.


#### Java

Two Java templates are provided `java11` and `java11-vertx`, both of which use Gradle as the build system. Please note that the `java8` template is deprecated, and should not be used.

> If you need a different version, then please fork the templates repository, or contact sales@openfaas.com to access additional templates via your OpenFaaS Premium Subscription.

Support is made available for external code repositories via the build.gradle file where you specify dependencies to fetch from repositories or JAR files to be added via the build.

* Write a function `java-function`:

```
$ faas-cli new --lang java11 java-function
```

* Write your code in:

./src/main/Handler.java

* Write `junit` tests in:

./src/tests/

* Update gradle config if needed in:

./build.gradle
./settings.gradle

* Working with headers (advanced)

You can view the code for the IRequest and IResponse in [the OpenFaaS templates-sdk](https://github.com/openfaas/templates-sdk/tree/master/java11/model/src/main/java/com/openfaas/model)

You can use `getHeader(k)` on the Request interface to query a header.

To set a header such as content-type you can use `setHeader(k, v)` on the Response interface.

You can also run the following to create a function using Vert.x

```bash
$ faas-cli new --lang java11-vertx java-vertx-function
```

#### CSharp / .NET Core 2.1

You can create functions in .NET Core 2.1 using C# / CSharp.

* Write a function named csharp-function

```
faas-cli new --lang csharp csharp-function
```

Now you can open your current folder in a tool such as Visual Studio Code and add dependencies using the project (csproj) file.

#### Ruby

Create a function called `ruby-function`:

```
$ faas-cli new --lang ruby ruby-function
```

The directory structure is:

```
├── ruby-function
│   ├── Gemfile
│   └── handler.rb
├── ruby-function.yml
```

Your code should be in the handler.rb file

##### Ruby: Adding a Gem (Library)

Open the `Gemfile` in the ruby-function directory

Add the following line

```Gemfile
gem 'httparty'
```

##### Ruby: Using our own Gem

Replace your `handler.rb` code with the following

```ruby
require 'httparty'

class Handler
    def run(req)
        return HTTParty.get("http://api.stackexchange.com/2.2/questions?site=stackoverflow&tagged=#{req}")
    end
end
```

##### Ruby: Building / Deploy / Run

Edit the `ruby-function.yml` and point your image to your dockerhub, for example
`${your_user}/ruby-function`

```
$ faas-cli up -f ruby-function.yml
...
Using bundler 1.16.4
Fetching multi_xml 0.6.0
Installing multi_xml 0.6.0
Fetching httparty 0.16.2
Installing httparty 0.16.2
Bundle complete! 1 Gemfile dependency, 3 gems now installed.
Bundled gems are installed into `/usr/local/bundle`
Post-install message from httparty:
When you HTTParty, you must party hard!
...
```

Now you can invoke the function:

```sh
$ echo 'OpenFaaS' | faas-cli invoke ruby-function
{
   "quota_remaining" : 298,
   "quota_max" : 300,
   "has_more" : false,
   "items" : [
      {
         "title" : "Scaling with GPU usage",
         "creation_date" : 1536315498,
         "answer_count" : 0,
         "view_count" : 10,
         "is_answered" : false,
...
```

#### Ruby HTTP

As an alternative to the `ruby` template, which uses the classic watchdog, we have an alternative where you can set HTTP response headers.

```
faas-cli template store pull ruby-http

faas-cli new --lang ruby-http k8s-get-pods
```

To add support for native dependencies such as kubeclient, you need to add the `dev` package to the `build_options`:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  k8s-get-pods:
    lang: ruby-http
    handler: ./k8s-get-pods
    image: k8s-get-pods:latest
    build_options:
    - dev
```

Then update your Gemfile:

```
source 'https://rubygems.org'

gem "kubeclient"
```

```
faas-cli build -f k8s-get-pods.yml
```

#### PHP7

To create a PHP7 function named `my-function` type in:

    $ faas-cli new my-function --lang php7

You'll see:

    my-function.yml
    my-function/src/Handler.php
    my-function/composer.json
    my-function/php-extension.sh

Add any dependencies/extensions as described below and implement your functions business logic in `Handler.php`.

#### PHP7 - Composer Dependencies

You should edit `composer.json` and add any required package dependencies, referring to the [Composer Documentation](https://getcomposer.org/doc/) for instructions on using `composer.json`.

##### PHP7 - Private Composer Repositories

Refer to the [PHP7 Template Documentation](https://github.com/openfaas/templates/tree/master/template/php7) for instructions on how to use [Composers](https://getcomposer.org/doc/) `COMPOSER_AUTH` environment variable to configure access to dependencies in private repositories.

##### PHP7 - PHP Extensions

The PHP7 template is based upon the [Docker Library PHP image](https://hub.docker.com/_/php/) and provides the `php-extension.sh` script which exposes the ability to customise extensions installed in a function image.

Refer to the [PHP7 Template Documentation](https://github.com/openfaas/templates/tree/master/template/php7) for instructions on customising installed extensions.

#### Customise a template

It is recommended that you use the official templates as they are provided and if there is a short-coming that you raise a GitHub issue so we can improve the templates for everyone.

All templates are driven by a Dockerfile and can be customised by editing the files found in the ./template folder.

##### Update the Dockerfile

There are several reasons why you may want to update your Dockerfile, just edit `./template/<language_name>/Dockerfile`.

* New base image - some companies prefer to use their own base images for Docker images for compliance, support or licensing reasons

* Add native package - sometimes you may want to add a native package from the Alpine Linux repository or the Debian package repository - just add a step into the Dockerfile

* Try a new version of a base-image - it may be that the project is showing support for Node.js LTS, but you want the cutting-edge version, you can do that too

##### Update a template's configuration

The name of a template is read from a "template.yml" file kept within the template folder: `./template/<language_name>/template.yml`

For `csharp` we have the following:

```
language: csharp
fprocess: dotnet ./root.dll
```

* `language` is the display name used for `faas-cli new --list`.
* `fprocess` provides the process to run for each invocation - i.e. your function

##### Use your own templates

You can use your own Git repository for a custom or forked set of templates. This can be public or private.

See `faas-cli template pull` for more information.

If you want to set up your own default template location, specify the `OPENFAAS_TEMPLATE_URL` environmental variable the following way:

```bash
export OPENFAAS_TEMPLATE_URL=https://raw.githubusercontent.com/user/mytemplate/customtemplates
```

##### Download templates from the template store

> Note: In order to access the template store you need `0.8.1` version of the CLI or higher

Check what templates are available in the template store with the CLI by typing:

```bash
faas-cli template store list
```

Pull the desired template by specifying `NAME` attribute only:

```bash
faas-cli template store pull go
```

or pull the template by mixing the `REPOSITORY` and `NAME` attributes the following way:

```bash
faas-cli template store pull openfaas/go
```

To get more information on specific store use the `describe` verb like:

```bash
faas-cli template store describe openfaas/go
```

or if there is no collision between names use only the name field:

```bash
faas-cli template store describe go
```

If you have your own store with templates, you can set that as your default official store by setting the environmental variable `OPENFAAS_TEMPLATE_STORE_URL` the following way:

```bash
export OPENFAAS_TEMPLATE_STORE_URL=https://raw.githubusercontent.com/user/openfaas-templates/templates.json
```

Now the source of the store is changed to the URL you have specified above.

## ARM / Raspberry Pi

It is possible to migrate to use multi-arch templates with OpenFaaS, feel free to ask the community for direction here.

Otherwise, for ARM and Raspberry Pi you will need to build on the device, and not on your PC or CI server.
