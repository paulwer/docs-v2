---
title: Optimize writes to InfluxDB
description: >
  Simple tips to optimize performance and system overhead when writing data to
  InfluxDB Clustered.
weight: 203
menu:
  influxdb_clustered:
    name: Optimize writes
    parent: write-best-practices
influxdb/cloud/tags: [best practices, write]
related:
  - /resources/videos/ingest-data/, How to Ingest Data in InfluxDB (Video)
---

Use these tips to optimize performance and system overhead when writing data to InfluxDB.

- [Batch writes](#batch-writes)
- [Sort tags by key](#sort-tags-by-key)
- [Use the coarsest time precision possible](#use-the-coarsest-time-precision-possible)
- [Use gzip compression](#use-gzip-compression)
- [Synchronize hosts with NTP](#synchronize-hosts-with-ntp)
- [Write multiple data points in one request](#write-multiple-data-points-in-one-request)

{{% note %}}
The following tools write to InfluxDB and employ _most_ write optimizations by default:

- [Telegraf](/influxdb/clustered/write-data/use-telegraf/)
- [InfluxDB client libraries](/influxdb/clustered/reference/client-libraries/)
{{% /note %}}

## Batch writes

Write data in batches to minimize network overhead when writing data to InfluxDB.

{{% note %}}
The optimal batch size is 10,000 lines of line protocol or 10 MBs,
whichever threshold is met first.
{{% /note %}}

## Sort tags by key

Before writing data points to InfluxDB, sort tags by key in lexicographic order.
_Verify sort results match results from the [Go `bytes.Compare` function](http://golang.org/pkg/bytes/#Compare)._

```sh
# Line protocol example with unsorted tags
measurement,tagC=therefore,tagE=am,tagA=i,tagD=i,tagB=think fieldKey=fieldValue 1562020262

# Optimized line protocol example with tags sorted by key
measurement,tagA=i,tagB=think,tagC=therefore,tagD=i,tagE=am fieldKey=fieldValue 1562020262
```

## Use the coarsest time precision possible

By default, InfluxDB writes data in nanosecond precision.
However if your data isn't collected in nanoseconds, there is no need to write at that precision.
For better performance, use the coarsest precision possible for timestamps.

_Specify timestamp precision when [writing to InfluxDB](/influxdb/clustered/write-data/)._

## Use gzip compression

Use gzip compression to speed up writes to InfluxDB.
Benchmarks have shown up to a 5x speed improvement when data is compressed.

{{< tabs-wrapper >}}
{{% tabs %}}
[Telegraf](#)
[Client libraries](#)
[InfluxDB API](#)
{{% /tabs %}}
{{% tab-content %}}

### Enable gzip compression in Telegraf

In the `influxdb_v2` output plugin configuration in your `telegraf.conf`, set the
`content_encoding` option to `gzip`:

```toml
[[outputs.influxdb_v2]]
  urls = ["https://{{< influxdb/host >}}"]
  # ...
  content_encoding = "gzip"
```

{{% /tab-content %}}
{{% tab-content %}}

### Enable gzip compression in InfluxDB client libraries

Each [InfluxDB client library](/influxdb/clustered/reference/client-libraries/) provides
options for compressing write requests or enforces compression by default.
The method for enabling compression is different for each library.
For specific instructions, see the
[InfluxDB client libraries documentation](/influxdb/clustered/reference/client-libraries/).
{{% /tab-content %}}
{{% tab-content %}}

### Use gzip compression with the InfluxDB API

When using the InfluxDB API `/api/v2/write` endpoint to write data,
compress the data with `gzip` and set the `Content-Encoding` header to `gzip`--for example:

{{% influxdb/custom-timestamps %}}
{{% code-callout "Content-Encoding: gzip" "orange" %}}
```bash
echo "mem,host=host1 used_percent=23.43234543 1641024000
mem,host=host2 used_percent=26.81522361 1641027600
mem,host=host1 used_percent=22.52984738 1641031200
mem,host=host2 used_percent=27.18294630 1641034800" | gzip > system.gzip \

curl --request POST "https://{{< influxdb/host >}}/api/v2/write?org=ignored&bucket=DATABASE_NAME&precision=s" \
  --header "Authorization: Token DATABASE_TOKEN" \
  --header "Content-Encoding: gzip" \
  --data-binary @system.gzip
```
{{% /code-callout %}}
{{% /influxdb/custom-timestamps %}}
{{% /tab-content %}}
{{< /tabs-wrapper >}}

## Synchronize hosts with NTP

Use the Network Time Protocol (NTP) to synchronize time between hosts.
If a timestamp isn't included in line protocol, InfluxDB uses its host's local
time (in UTC) to assign timestamps to each point.
If a host's clocks isn't synchronized with NTP, timestamps may be inaccurate.

## Write multiple data points in one request

To write multiple lines in one request, each line of line protocol must be delimited by a new line (`\n`).
