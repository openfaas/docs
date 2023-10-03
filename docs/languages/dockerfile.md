## Dockerfile language

The Dockerfile language is treated as a special case for `faas-cli`, instead of building with a pre-defined entrypoint abstracted from the user, the whole contents of a given folder are used to build an image.

The default Dockerfile shows how to fork a CLI process once per request, however you can put whatever you like in the Dockerfile.

The example uses classic-watchdog, however the of-watchdog and existing Dockerfiles are supported too. Just make sure that whatever you use conforms to the [OpenFaaS Workloads defintion](/reference/workloads).

### Turn a CLI into a function

The following example shows how to turn a CLI into a function:

```bash
export OPENFAAS_PREFIX=ttl.sh/test

faas-cli new --lang dockerfile curl
```

Inspect the contents of `curl/Dockerfile`:

```
FROM ghcr.io/openfaas/classic-watchdog:0.2.1 as watchdog

FROM alpine:3.18

RUN mkdir -p /home/app

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

# Add non root user
RUN addgroup -S app && adduser app -S -G app
RUN chown app /home/app

WORKDIR /home/app

USER app

# Populate example here - i.e. "cat", "sha512sum" or "node index.js"
ENV fprocess="cat"
# Set to true to see request in function logs
ENV write_debug="false"

EXPOSE 8080

HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]
```

To fork `curl` for every HTTP request, add it to the Alpine Linux container:

```diff
+RUN apk add --no-cache curl
```

Next, change `fprocess` to: `xargs curl -s`.

The `xargs` helper will take the request body and convert it into arguments for `curl`. 

```diff
-ENV fprocess="cat"
+ENV fprocess="xargs curl -s -L -S"
```

Build and deploy the function:

```bash
faas-cli local-run -f curl.yml

curl -s http://127.0.0.1 -d "https://www.openfaas.com"
```

### Explore HTTP request headers

Create a new function called `env`:

```bash
export OPENFAAS_PREFIX=ttl.sh/test
faas-cli new --lang dockerfile env
```

Now edit `env/Dockerfile` and change the `fprocess` to `env`:

```diff
-ENV fprocess="cat"
+ENV fprocess="env"
```

Test the function:

```bash
faas-cli local-run -f env.yml

curl -s http://127.0.0.1:8080 -d test
```

You'll see the HTTP headers were made available as environment variables:

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=env-648d57f457-xtvwg
OPENFAAS_NAME=env
fprocess=env
HOME=/home/app
Http_X_Forwarded_Proto=https
Http_Accept=text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Http_Sec_Fetch_User=?1
Http_X_Call_Id=139138c6-6851-4902-ac98-dc589cf8a4c0
Http_Upgrade_Insecure_Requests=1
Http_X_Forwarded_For=10.42.0.6
Http_X_Forwarded_Host=openfaas.o6s.io
Http_X_Start_Time=1696329981927270613
Http_Sec_Ch_Ua="Google Chrome";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Http_Sec_Ch_Ua_Platform="Linux"
Http_Sec_Fetch_Dest=document
Http_Accept_Language=en-GB,en-US;q=0.9,en;q=0.8
Http_Sec_Fetch_Site=none
Http_X_Scheme=https
Http_User_Agent=Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36
Http_Accept_Encoding=gzip, deflate, br
Http_X_Forwarded_Scheme=https
Http_X_Real_Ip=10.42.0.6
Http_X_Request_Id=84c840c822ddcc346194db540e1c3eb2
Http_Sec_Ch_Ua_Mobile=?0
Http_Sec_Fetch_Mode=navigate
Http_X_Forwarded_Port=443
Http_Method=GET
Http_ContentLength=0
Http_Content_Length=0
Http_Path=/
Http_Host=10.42.0.71:8080
```

These can be accessed by any process that is executed for each request when using the [classic watchdog](https://github.com/openfaas/classic-watchdog/).

