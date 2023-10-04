# Create new functions

## Get started

Once you've installed the `faas-cli` you can start creating and deploying functions via the `faas-cli up` command or using the individual commands:

* `faas-cli build` - build an image into the local Docker library
* `faas-cli push` - push that image to a remote container registry
* `faas-cli deploy` - deploy your function into a cluster

The `faas-cli up` command automates all of the above in a single command.

For Raspberry Pi and Arm, you must use the `publish` command instead of `build` and `push`, or `up`.

See the notes here: [Building multi-arch images for Arm and Raspberry Pi](/cli/build/)

!!! info "Information about OpenFaaS templates is moving"

    The guide for OpenFaaS templates is moving to the ["Languages" section"](/languages/overview.md) in its own section.

## Templates

The OpenFaaS CLI has a template engine built-in which can create new functions in a given programming language. The way this works is by reading a list of templates from the `./template` location in your current working folder.

Before creating a new function make sure you pull in the official OpenFaaS language templates from GitHub via the [templates repository](https://github.com/openfaas/templates).

```bash
$ faas-cli template pull
```

This page shows how to generate functions in the most popular languages and explains how you can manage their dependencies too.

### Classic vs. of-watchdog templates

The *Classic Templates* are held in the [openfaas/templates](https://github.com/openfaas/templates) repository and are based upon the *Classic Watchdog* which uses STDIO to communicate with your function. The of-watchdog uses HTTP to communicate with functions and most of its templates are available in the [openfaas](https://github.com/openfaas/) organisation in their own separate repositories on GitHub and in the store.

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
ruby-http               openfaas-incubator Ruby 2.4 HTTP template
golang-middleware       openfaas Golang Middleware template
csharp-httprequest      distantcam         C# HTTP template
...
```

Choose a template and retrieve it locally with the command:

```sh
$ faas-cli template store pull csharp
```

Once downloaded, your chosen template and any others stored in the same repository will be available to use:

```sh
$ faas-cli new --list
Languages available as templates:
- csharp
```

You can add your own store just by specifying the `--url` flag for both commands to pull and list your custom templates store.

The classic templates are held in the [openfaas/templates](https://github.com/openfaas/templates) repository.

#### Go templates

See: [Go template](/languages/go)

#### Python 3 templates

See: [Python template](/languages/python)

#### Node.js templates (of-watchdog template)

See: [Node template](/languages/node)

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

## Arm / Raspberry Pi

It is possible to migrate to use multi-arch templates with OpenFaaS, feel free to ask the community for direction here.

Otherwise, for Arm and Raspberry Pi you will need to build on the device, and not on your PC or CI server.
