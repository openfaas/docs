# Function Logs

Via an extensible API, OpenFaaS provides access to function logs via the Gateway API and the CLI. This provides a simple and consistent way to access function logs regardless of your orchestration provider.

The log provider for Kubernetes `faas-netes` fetches records directly from the Kubernetes API, so it will give similar output to `kubectl logs -n openfaas-fn deploy/function`.

The `faas-cli logs NAME` command will stream the logs for the named function.  By default, it will attempt to follow the logs, but you can control the behavior of the stream using these flags

```sh
  -g, --gateway string           Gateway URL starting with http(s):// (default "http://127.0.0.1:8080")
  -h, --help                     help for logs
      --instance                 print the function instance name/id
      --lines int                number of recent log lines file to display. Defaults to -1, unlimited if <=0 (default -1)
      --name                     print the function name
  -n, --namespace string         Namespace of the function
  -o, --output logformat         output logs as (plain|keyvalue|json), JSON includes all available keys
      --since duration           return logs newer than a relative duration like 5s
      --since-time timestamp     include logs since the given timestamp (RFC3339)
  -t, --tail                     tail logs and continue printing new logs until the end of the request, up to 30s (default true)
      --time-format timeformat   string format for the timestamp, any value go time format string is allowed, empty will not print the timestamp (default 2006-01-02T15:04:05Z07:00)
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

You can also get logs in JSON format:

```bash
$ faas-cli logs trove --format json

{"name":"trove","namespace":"openfaas-fn","instance":"9174","timestamp":"2021-02-12T17:01:03.088068Z","text":"User requested \"Insiders Update: 1st Feb 2020 - Java, KubeCon, Istio, Crossplane and more!\""}
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

Log retention and history will be determined by your cluster configuration and the log provider installed. The default configuration in the function provider for Kubernetes streams logs directly from the cluster. This means that you will only see logs from running function containers, and no long term history is not available. If you delete a function, you remove access to its logs.

With faasd, logs are stored in the [journal](https://wiki.archlinux.org/index.php/Systemd/Journal), so the retention period lasts beyond the life of a function, however it may be limited by time or the number of lines.

If you want to retain logs over time, consider using a log aggregator like [Elastic Search](https://www.elastic.co/), [Logz.io](https://logz.io) or [Humio](https://www.humio.com) as an external data-store.

[Grafana Loki](https://grafana.com/oss/loki/) can also provide fast access to logs without much additional overhead on the cluster. A log provider exists to enable "faas-cli" and the OpenFaaS API to query an external data-store, instead of Kubernetes itself.

## Alternative Log Providers

The log system is designed to be extended with alternative providers, this means that logs could instead be supplied by a persistent storage, e.g. Loki or ElasticSearch.  See the [logs provider overview](../architecture/logs-provider.md) for more details about how providers work and available alternatives.

## Structured Logs

Structured logs are a form of machine-readable logs that treats logs as data sets rather than text which allows logs to be more easily searched and analyzed. Typically, these will be in the form of a JSON object on a single line and will allow you to achieve a high level of granular detail.

Introduced in of-watchdog v0.8.2, you can set the `prefix_logs` environment variable to `false`. This will remove the log prefix for all messages received via stdout/stderr, meaning that only the `<msg>` will be sent to the terminal.
