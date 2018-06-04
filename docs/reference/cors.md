# Dealing with CORS

> No 'Access-Control-Allow-Origin' header is present on the requested resource

If you have seen the error message above in your web application, this is the correct page to have a look at. This error is related to a topic called Cross-Origin Resource Sharing (CORS).

## Understanding the Same-origin policy

First of all, you need to know that browsers have a policy called the **Same-origin policy** for security reasons. Under this policy, a script inside a web page can access other web page's data only if they both have the same **origin**. An **origin** consists of URI scheme, host name, and port number. This means, a script inside `http://example.com` can only access data within the same origin such as `http://example.com/users`. Trying to access data in other origins such as `http://sample.com` will lead to a violation error.





You don't want arbitrary scripts sending to or retrieving data from malicious web sites so the **Same-origin policy** plays a very important role in order to protect you from these threats. 

## Problems in Developing Web Applications Locally

There are situations where the **Same-origin policy** gets in the way. What if you wanted to develop a Single Page Application locally and access data via OpenFaaS functions? Often, developers get stuck with an error telling you that you don't have a `Access-Control-Allow-Origin` header present. This is because your local web server and the OpenFaaS gateway have **different origins** as illustrated below.


## Using Reverse Proxies as a Workaround

Mainly there are two ways to deal with this problem. Either set a CORS header (which is not covered in this page) or make the browser **think** it is talking to the same origin using reverse proxies. If you place a reverse proxy inside or in front of your web server, you can route specific requests to specific targets. For example, you can route all ajax requests in your web application which start with `/api` to the OpenFaaS gateway by proxying it from the web server as illustrated below.


This way, the request from the browser doesn't violate the **Same-origin policy** and under the hood, would be proxied to the OpenFaaS gateway.

## An Example with a Vue.js + Webpack Application