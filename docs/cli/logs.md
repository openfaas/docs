# Function Logs

OpenFaaS provides access to function logs via an API and the CLI: `faas-cli logs NAME`.

This command will stream the logs for the named function.  By default, it will attempt to follow the logs, but you can control the behavior of the stream using these flags

```sh
--follow                 continue printing new logs until the end of the request, up to 30s (default true)
--since duration         return logs newer than a relative duration like 5s
--since-time timestamp   include logs since the given timestamp (RFC3339)
--tail int               number of recent log lines file to display. Defaults to -1, unlimited if <=0 (default -1)
```

## Log structure
The logs for a function will look

```
<RCF8601 Timestamp> <function name> (<container instance>) <msg>
```

where `msg` is the container logs, this typically contains stdout and stderr of the _contianer_.

An example log output for `nodeinfo` from the function store is

```sh
2019-07-21 07:57:14.437219758 +0000 UTC nodeinfo (nodeinfo-867cc95845-p9882) 2019/07/21 07:57:14 Wrote 92 Bytes - Duration: 0.121959 seconds
```

## Filter and Search

The CLI writes logs to stdout, so it can easily be chained with any of your favorite CLI tools: `grep`, `sed`, [`fzf`](https://github.com/junegunn/fzf) etc.

For example, the Sentiment Analysis function in the Store prints a version string during startup.

```sh
faas-cli store deploy SentimentAnalysis
faas-cli logs sentimentanalysis --follow=false | grep "Version"
2019-07-21 08:25:27.684193321 +0000 UTC sentimentanalysis (sentimentanalysis-76bd68b8b-529dk) 2019/07/21 08:25:27 Version: 0.13.0	SHA: fa93655d90d1518b04e7cfca7d7548d7d133a34e
```

## Log Retention and History
Log retention and history will be determined by your cluster configuration and the log provider installed. The default configuration in the official function providers (`faas-swarm` and `faas-netes`) stream logs directly from the cluster containers. This means that you will only see logs from running function containers, no long term history.  So deleting a function will also remove access to those logs.

The log system is designed to be extended with alternative providers, this means that logs could instead be supplied by a persistent storage, e.g. Loki or ElasticSearch.  See the [log providers overview](../reference/logs/providers.md) for more details about how providers work and available alternatives.






