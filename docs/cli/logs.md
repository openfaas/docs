# Function Logs

Via an extensible API, OpenFaaS provides access to function logs via the Gateway API and the CLI. This provides a simple and consistent way to access function logs regardless of your orchestration provider.


The official `faas-swarm` and `faas-netes` providers stream logs directly from the cluster API, this means that you will get the same logs as when you use `docker service logs` and `kubectl logs`. OpenFaaS only wraps wraps the existing container-native log systems, so you can always access function logs via the orchestration CLIs.

The `faas-cli logs NAME` command will stream the logs for the named function.  By default, it will attempt to follow the logs, but you can control the behavior of the stream using these flags

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

For example, when you set the `write_debug` environment variable in the Sentiment Analysis function from the Store, it will print the function output to the logs, we can then search that output using grep.  For example,

```sh
faas-cli store deploy SentimentAnalysis --env write_debug=true
echo "i like code" | faas-cli invoke sentimentanalysis
echo "i like functions" | faas-cli invoke sentimentanalysis
echo "i like containers" | faas-cli invoke sentimentanalysis
faas-cli logs sentimentanalysis | grep sentence_count
2019-07-22 12:44:08.202436478 +0000 UTC sentimentanalysis (sentimentanalysis-7887c5d8c5-5rnb5) {"polarity": 0.0, "sentence_count": 1, "subjectivity": 0.0}
2019-07-22 12:44:10.11422064 +0000 UTC sentimentanalysis (sentimentanalysis-7887c5d8c5-5rnb5) {"polarity": 0.0, "sentence_count": 1, "subjectivity": 0.0}
2019-07-22 12:44:11.882708263 +0000 UTC sentimentanalysis (sentimentanalysis-7887c5d8c5-5rnb5) {"polarity": 0.0, "sentence_count": 1, "subjectivity": 0.0}
```

## Log Retention and History
Log retention and history will be determined by your cluster configuration and the log provider installed. The default configuration in the official function providers (`faas-swarm` and `faas-netes`) stream logs directly from the cluster containers. This means that you will only see logs from running function containers, no long term history.  So deleting a function will also remove access to those logs.

## Alternative Log Providers
The log system is designed to be extended with alternative providers, this means that logs could instead be supplied by a persistent storage, e.g. Loki or ElasticSearch.  See the [log providers overview](../reference/logs/providers.md) for more details about how providers work and available alternatives.

