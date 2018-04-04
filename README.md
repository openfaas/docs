# OpenFaaS docs repository

This is the source repository for the OpenFaaS documentation site.

The OpenFaaS docs generated from this site are not yet launched or live.

For local development:

```shell
# docker run --rm -it -p 8000:8000 -v `pwd`:/docs squidfunk/mkdocs-material
```

## Published page

This page is published through the use of `mkdocs` and is hosted on https://netlify.com/ with an SSL cert from LetsEncrypt.

* https://docs.openfaas.com/

All commits into master (or merged PRs) will appear on the front-page after being rebuilt.

## mkocs-material Extensions

To view the special formatting extensions avaialble for these docs read the mkdoc-material documentation [here](https://squidfunk.github.io/mkdocs-material/extensions/admonition/)