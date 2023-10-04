## PHP 8.2

The `php8` template currently uses PHP 8.2 along with Alpine Linux.

It is a forking template which uses the [Classic Watchdog](https://github.com/openfaas/classic-watchdog) to fork a PHP process for each and every HTTP request.

Dependencies for functions are managed with [Composer](https://getcomposer.org/).

This template is managed and maintained by community members.

### Create a new function

To create a function using PHP 8.2 named `my-php` type in:

```bash
faas-cli new --lang php8 \
    my-php
```

This will create several new files:

* my-php.yml - the function's YAML file
* my-php/src/Handler.php - the function's handler for reading the input and writing the output
* my-php/composer.json - for managing dependencies
* my-php/php-extension.sh - for installing PHP extensions

Add any dependencies/extensions as described below and implement the handler in `Handler.php`.

```php
<?php

namespace App;

/**
 * Class Handler
 * @package App
 */
class Handler
{
    /**
     * @param string $data
     * @return string
     */
    public function handle(string $data): string
    {
        return $data;
    }
}
```

The input and output of the function is a string.

### Adding a Composer dependency

You should edit `composer.json` within the function's handler folder, and add any required package dependencies, referring to the [Composer Documentation](https://getcomposer.org/doc/) for instructions on using `composer.json`.

### Private Composer Repositories

Refer to the [PHP Template Documentation](https://github.com/openfaas/templates/tree/master/template/php8) for instructions on how to use [Composers](https://getcomposer.org/doc/) `COMPOSER_AUTH` environment variable to configure access to dependencies in private repositories.

In some cases, you may need to use private composer repositories - using the `faas-cli` you can pass in
a build argument during build, for example;

```
faas-cli build -f ./functions.yml \
  --build-arg COMPOSER_AUTH='{"bitbucket-oauth": {"bitbucket.org": {"consumer-key": "xxxxxxxx","consumer-secret": "xxxxxxx"}}}'
```

See more information [here](https://getcomposer.org/doc/05-repositories.md#git-alternatives).

That way you can pass in tokens for Composer, if necessary, GitHub tokens to get around rate-limit issues.

Bear in mind that any tokens used with `--build-arg` will be made available in the final container image.

[OpenFaaS Standard's](https://openfaas.com/pricing) `faas-cli pro build` has a specific way to handle this without leaking secrets into the final image.

### PHP Extensions

The PHP template is based upon the [Docker Library PHP image](https://hub.docker.com/_/php/) and provides the `php-extension.sh` script which exposes the ability to customise extensions installed in a function image.

The following modules are preinstalled within the function:

| Modules |
| ------------- |
| Core, date, libxml, openssl, pcre, sqlite3, zlib, ctype, curl, dom, fileinfo, filter, ftp, hash, iconv, json, mbstring, SPL, PDO, pdo_sqlite, session, posix, readline, Reflection, standard, SimpleXML, Phar, tokenizer, xml, xmlreader, xmlwriter, mysqlnd, sodium |

If you need to install [Phalcon](https://github.com/phalcon) for example, check out the
following sample which you could use in your functions `src/php-extension.sh` file;

```bash
#!/bin/sh

echo "Installing PHP extensions"
docker-php-ext-install pdo_mysql

# Install Phalcon
PHALCON_VERSION=3.4.0
PHALCON_EXT_PATH=php7/64bits

set -xe && \
        # Compile Phalcon
        curl -LO https://github.com/phalcon/cphalcon/archive/v${PHALCON_VERSION}.tar.gz && \
        tar xzf ${PWD}/v${PHALCON_VERSION}.tar.gz && \
        docker-php-ext-install -j $(getconf _NPROCESSORS_ONLN) ${PWD}/cphalcon-${PHALCON_VERSION}/build/${PHALCON_EXT_PATH} && \
        # Remove all temp files
        rm -r \
            ${PWD}/v${PHALCON_VERSION}.tar.gz \
            ${PWD}/cphalcon-${PHALCON_VERSION}
```

You can also refer to the PHP Docker image [documentation](https://github.com/docker-library/docs/blob/master/php/README.md#how-to-install-more-php-extensions) for additional instructions on the installation and configuration of extensions
