# OpenFaaS docs repository

This is the source repository for the OpenFaaS documentation site.

For local development:

```shell
# docker run --rm -it -p 8000:8000 -v `pwd`:/docs squidfunk/mkdocs-material
```

## Published page

This page is published through the use of `mkdocs` and is hosted on https://netlify.com/ with an SSL cert from LetsEncrypt.

* https://docs.openfaas.com/

All commits into master (or merged PRs) will appear on the front-page after being rebuilt.

## mkdocs-material markdown extensions

There are several markdown extensions that can be used to create special formatting. Look at the docs [here](https://squidfunk.github.io/mkdocs-material/extensions/admonition/) for all available extensions.

## Adding OpenFaaS Users

The list of OpenFaaS users can be found within [docs/index.md](docs/index.md#users-of-openfaas).  Additions to this list should be made while maintaining the alphabetical ordering.

