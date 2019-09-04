# User guide for OpenFaaS Cloud

This guide applies to:

* [OpenFaaS Cloud: Self-hosted](/openfaas-cloud/)
* [OpenFaaS Cloud: The Community Cluster](/openfaas-cloud/community-cluster/)

## The guide

We'll be creating a Node.js API or microservice using the [node10-express template](https://github.com/openfaas-incubator/node10-express-template). It does a HTTP redirect for a number of hard-coded routes and could be extended into a fully-functional url shortener in the future.

You can use any of the official OpenFaaS templates, the incubator templates for Node.js and Golang, or even your own custom templates. 

If you have the `dockerfile` language enabled, you can deploy an API or web application which can be packaged in a Docker image.

### Create a test repo

First of all create a test repo in the GitHub UI, call it `goto`.

![Create a repo](/images/openfaas-cloud/welcome-01.png)

This can be public or private.

### Install the GitHub App

Now find your GitHub App URL. If you're using The Community Cluster, [you will find that here](/openfaas-cloud/community-cluster).

Otherwise, click Settings -> Developer Settings -> GitHub Apps. Find yours and click "Edit" and then "Install App"

![](/images/openfaas-cloud/welcome-02.png)

Pick the account you want to install it on, it should match the organisation or username where you created your repo.

Install the GitHub App on a single repository, or if you have multiple, check each of them:

![](/images/openfaas-cloud/welcome-03.png)

Your GitHub repo will now send webhooks to OpenFaaS Cloud whenever:

* there is a `git push` event
* or whenever you delete a repo or remove the integration

### Clone the repo and add the source code

* Clone your repo

```
git clone https://github.com/alexellis/goto
cd goto
```

* Pull in the OpenFaaS template for node10 and Express.js:

```sh
faas-cli template store pull node10-express
```

You can find other templates with `faas-cli template store list`

* Create a new API

```sh
faas-cli new --lang node10-express goto
```

* Rename its YAML file to `stack.yml`

```sh
mv goto.yml stack.yml
```

Now add some code to do a redirect into `goto/handler.js`.

* If we hit `/` then we'll print an error
* If we hit `/home/` then we'll send the user to my homepage.

```sh
"use strict"

module.exports = (event, context) => {
    let redirect;

    if(event.path == "/home") {
        redirect = "https://www.alexellis.io/";
    } else if(event.path == "/sponsors" || event.path == "/insiders") {
        redirect = "https://github.com/users/alexellis/sponsorship";
    }

    process.stderr.write(event.path)

    if(!redirect) {
        return context
            .status(400)
            .fail("Unknown short URL");
    }
    context
        .status(302)
        .headers({"location": redirect})
        .succeed();
}
```

Now rather than having to build this or test it locally, we can push it straight up to GitHub.

```sh
git add .
git commit -s -m "Initial commit"
git push origin master
```

### Check the status of the build

Now check the status of the build by viewing the Commits page.

[https://github.com/alexellis/goto/commits/](https://github.com/alexellis/goto/commits/)

You will see your build queued up:

![](/images/openfaas-cloud/welcome-04.png)

Then in a few moments, you should see a pass or failure and a link you can click to find out more.

![](/images/openfaas-cloud/welcome-05.png)

The detailed view will provide any logs that are available from the build, this would include unit test failures or linting errors.

![](/images/openfaas-cloud/welcome-06.png)

### Checkout the dashboard

Head over to your dashboard and prepare to log into GitHub with 2FA

Go to:

```
https://system.example.com/dashboard/username
```

Replace `example.com` with the OFC installation, i.e. `o6s.io` for The Community Cluster.

* Enter your 2FA details if you have that enabled

![](/images/openfaas-cloud/welcome-08.png)

* Authorize the OAuth login

![](/images/openfaas-cloud/welcome-07.png)

* Checkout the overview page

You have a complete list of your functions available here

![](/images/openfaas-cloud/welcome-09.png)

* Checkout the details page

Click on one of the functions to view its details page.

![](/images/openfaas-cloud/welcome-09.png)

From here you can get metrics, find the latest diff, Docker image artifact and see how many replicas of the function are active.

Get the *Endpoint* for your API and then open it in a browser.

* `https://alexellis.cloudnative.space/goto/` - generates an error
* `https://alexellis.cloudnative.space/goto/home` - redirects to home

The default behaviour with no short-path is to return an error, and the success behaviour is to redirect the browser to my homepage. So you can see after visiting these two URLs, we have a 50/50 split.

### Next steps

It is likely that you will need to add a secret or some confidential data to your API or function. 

* Read the [User Guide for Secrets](/openfaas-cloud/secrets/) for OpenFaaS Cloud.

* [Chat with the community on Slack](/community/)