# Introduction

Personal notes on running Loki, Promtail, and Grafana in an on-prem big server scenario.

Tested with v2.8.4

https://github.com/grafana/loki/releases/

## General notes

Try to emulate zap's logging, e.g. an ``Info`` line: https://pkg.go.dev/go.uber.org/zap#Logger.Info

All log lines need a ``message``.

## Launching

```
# Cassandra 4.1.3
./bin/cassandra -f

# Loki. Need to use table-manager otherwise it won't create tables in Cassandra
./loki-linux-amd64 -config.file=loki-local-config.yaml -target=all,table-manager 

# Promtail
./promtail-linux-amd64 -config.file=promtail-local-config.yaml

# 10.0.0 
./bin/grafana-server 
```

## Cassandra

TODO

## Loki

Loki has a REST API for queries. 

The query language is LogQL: https://megamorf.gitlab.io/cheat-sheets/loki/

A malformed query string will often result in the bewildering error 
["Invalid Numeric Literal" on colon following a single-quoted string (incorrect parser error message)](https://github.com/jqlang/jq/issues/501)
from curl (it uses jq under the hood?).

Examples below show how to quote strings, regexes, etc.

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/labels" | jq
{
  "status": "success",
  "data": [
    "filename",
    "job",
    "level"
  ]
}
```

Query a Loki instance, filter on a key/value in the json:

```
curl -s -G "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="onpremlogs"} | json |= `"key with spaces": "so silly"`' \
  --data-urlencode 'since=48h'  | jq
{
  "status": "success",
  "data": {
    "resultType": "streams",
    "result": [
      {
        "stream": {
          "caller": "zap/main.go:42",
          "filename": "/scratch/onpremsystemlogs/foo-2023-08-19.log",
          "job": "onpremlogs",
          "key_with_spaces": "so silly",
          "level": "info",
          "level_extracted": "info",
          "message": "new line 55",
          "timestamp": "2023-08-20T01:08:06.333000"
        },
        "values": [
          [
            "1692493686333000000",
            "{\"level\":\"info\",\"timestamp\":\"2023-08-20T01:08:06.333000\",\"caller\":\"zap/main.go:42\",\"message\":\"new line 55\", \"key with spaces\": \"so silly\"}"
          ]
        ]
      }
    ],
    "stats": {
      "summary": {
        "bytesProcessedPerSecond": 5497,
        "linesProcessedPerSecond": 42,
        "totalBytesProcessed": 3909,
        "totalLinesProcessed": 30,
        "execTime": 0.711054,

...
```

Find a string anywhere in the json:

```
$ curl -s -G "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="onpremlogs"} | json |= "so silly"' \
  --data-urlencode 'since=48h'  | jq
{
  "status": "success",
  "data": {
    "resultType": "streams",
    "result": [
      {
        "stream": {
          "caller": "zap/main.go:42",
          "filename": "/scratch/onpremsystemlogs/foo-2023-08-19.log",
          "job": "onpremlogs",
          "key_with_spaces": "so silly",
          "level": "info",
          "level_extracted": "info",
          "message": "new line 55",
          "timestamp": "2023-08-20T01:08:06.333000"
        },
        "values": [
          [
            "1692493686333000000",
            "{\"level\":\"info\",\"timestamp\":\"2023-08-20T01:08:06.333000\",\"caller\":\"zap/main.go:42\",\"message\":\"new line 55\", \"key with spaces\": \"so silly\"}"
          ]
        ]
      }
    ],
```

Use a regex to match the value in a field, must end with "silly":

```
curl -s -G "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="onpremlogs"} | json |~ `"key with spaces": ".*silly"`' \
  --data-urlencode 'since=48h'  | jq
{
  "status": "success",
  "data": {
    "resultType": "streams",
    "result": [
      {
        "stream": {
          "caller": "zap/main.go:42",
          "filename": "/scratch/onpremsystemlogs/foo-2023-08-19.log",
          "job": "onpremlogs",
          "key_with_spaces": "so silly",
          "level": "info",
          "level_extracted": "info",
          "message": "new line 55",
          "timestamp": "2023-08-20T01:08:06.333000"
        },
        "values": [
          [
            "1692493686333000000",
            "{\"level\":\"info\",\"timestamp\":\"2023-08-20T01:08:06.333000\",\"caller\":\"zap/main.go:42\",\"message\":\"new line 55\", \"key with spaces\": \"so silly\"}"
          ]
        ]
      }
    ],
```

## Promtail

Dry run to verify Promtail log parsing:

```bash
cat test.log |  ./promtail-linux-amd64 -config.file=promtail-local-config.yaml --stdin --dry-run --inspect --client.url http://127.0.0.1:3100/loki/api/v1/push

Clients configured:
----------------------
url: http://localhost:3100/loki/api/v1/push
batchwait: 1s
batchsize: 1048576
follow_redirects: false
enable_http2: false
backoff_config:
  min_period: 500ms
  max_period: 5m0s
  max_retries: 10
timeout: 10s
tenant_id: ""
drop_rate_limited_batches: false
stream_lag_labels: ""

----------------------
url: http://127.0.0.1:3100/loki/api/v1/push
batchwait: 1s
batchsize: 1048576
follow_redirects: false
enable_http2: false
backoff_config:
  min_period: 500ms
  max_period: 5m0s
  max_retries: 10
timeout: 10s
tenant_id: ""
drop_rate_limited_batches: false
stream_lag_labels: ""

[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.913648252 +0800 +08
	+: 2023-08-19 05:04:05 +0000 UTC
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.913649841 +0800 +08
	+: 2023-08-19 05:05:05 +0000 UTC
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-19T05:04:05+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-19T05:04:05.000000","caller":"zap/main.go:42","message":"new line 88888"}
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-19T05:05:05+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-19T05:05:05.000000","caller":"zap/main.go:42","message":"new line 99999"}
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.913653217 +0800 +08
	+: 2023-08-19 07:05:05 +0000 UTC
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.913657839 +0800 +08
	+: 2023-08-18 15:04:05 +0000 UTC
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-19T07:05:05+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-19T07:05:05.000000","caller":"zap/main.go:42","message":"new line 00011"}
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-18T15:04:05+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-18T15:04:05.000000","caller":"zap/main.go:42","message":"new line 22222"}
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.913761712 +0800 +08
	+: 2023-08-19 07:05:05 +0000 UTC
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.914551298 +0800 +08
	+: 2023-08-19 11:05:05 +0000 UTC
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-19T07:05:05+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-19T07:05:05.000000","caller":"zap/main.go:42","message":"new line 22222"}
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-19T11:05:05+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-19T11:05:05.000000","caller":"zap/main.go:42","message":"new line 3"}
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.915649056 +0800 +08
	+: 2023-08-19 11:05:06.333 +0000 UTC
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.916032552 +0800 +08
	+: 2023-08-19 11:08:06.333 +0000 UTC
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-19T11:05:06.333+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-19T11:05:06.333000","caller":"zap/main.go:42","message":"new line 444444"}
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-19T11:08:06.333+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-19T11:08:06.333000","caller":"zap/main.go:42","message":"new line 55", "some.other.crap": "error oh my gawd"}
[inspect: timestamp stage]: 
{stages.Entry}.Entry.Entry.Timestamp:
	-: 2023-08-20 09:48:14.916438338 +0800 +08
	+: 2023-08-20 01:08:06.333 +0000 UTC
[inspect: labels stage]: 
{stages.Entry}.Entry.Labels:
	-: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs"}
	+: {__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}
2023-08-20T01:08:06.333+0000	{__path__="/scratch/onpremsystemlogs/*.log", job="onpremlogs", level="info"}	{"level":"info","timestamp":"2023-08-20T01:08:06.333000","caller":"zap/main.go:42","message":"new line 55", "key with spaces": "so silly"}
```

Can see the timestamps being parse. Should not see a red ``none`` line for the timestamp stage.

Promtail timestamp formats are the known Go formats: https://pkg.go.dev/time

Notes on json parsing: https://community.grafana.com/t/i-cant-for-the-life-of-me-figure-out-how-to-parse-json-timestamp-in-promtail-config-yml/76504/2

# TODO

## Old logs

Set maximum old log to ingest?

https://grafana.com/docs/loki/latest/configuration/#ingester

https://github.com/grafana/loki/issues/5928#issuecomment-1102485160

## TMP, TEMP, ...

Adjust TMP for Loki, Promtail too?

## Retension policy

(Using "index" as I would in an ELK stack.)

How to see size of "indexes"? 

How to dump table sizes in Cassandra?

How to force loki to purge old indexes?

## Labels

Should add labels for host names (in the on-prem case of a handful of large servers).

Labels for service1, service2, etc. And another that covers everything.

Should we have request ID in a label for filtering? Otherwise client-side deals with too much data?

https://grafana.com/docs/loki/latest/best-practices/

> Things like, host, application, and environment are great labels. They will be fixed for a given system/app and have bounded values. Use static labels to make it easier to query your logs in a logical sense (e.g. show me all the logs for a given application and specific environment, or show me all the logs for all the apps on a specific host).

Short answer, no, it should not.

> In Loki 1.6.0 and newer the logcli series command added the --analyze-labels flag specifically for debugging high cardinality labels:
> 
> Total Streams:  25017
> Unique Labels:  8
> 
> Label Name  Unique Values  Found In Streams
> requestId   24653          24979
> logStream   1194           25016
> logGroup    140            25016
> accountId   13             25016
> logger      1              25017
> source      1              25016
> transport   1              25017
> format      1              25017
> 
> In this example you can see the requestId label had a 24653 different values out of 24979 streams it was found in, this is bad!!
> 
> This is a perfect example of something which should not be a label, requestId should be removed as a label and instead filter expressions should be used to query logs for a specific requestId. For example if requestId is found in the log line as a key=value pair you could write a query like this: {logGroup="group1"} |= "requestId=32422355"


https://stackoverflow.com/a/76670034

> Never use labels on something that could have unpredictable number of values (such as ``user_id`` or Request IP).
