---
title: Influx CLI
seotitle: Write data with the influx CLI
list_title: Write data with the influx CLI
weight: 5
description: >
Write data to InfluxDB using the `influx` CLI.
aliases:
menu:
  v2_0:
    name : Influx CLI
    parent: Write data
---

From the command line, use the [`influx write` command](/v2.0/reference/cli/influx/write/) to write data to InfluxDB.
Include the following in your command:

| Requirement          | Include by                                                                                         |
|:-----------          |:----------                                                                                         |
| Organization         | Use the `-o`,`--org`, or `--org-id` flags.                                                         |
| Bucket               | Use the `-b`, `--bucket`, or `--bucket-id` flags.                                                  |
| Precision            | Use the `-p`, `--precision` flag.                                                                  |
| Authentication token | Set the `INFLUX_TOKEN` environment variable or use the `t`, `--token` flag.                        |
| Data                 | Write data using **line protocol** or **annotated CSV**. Pass a file with the `-f`, `--file` flag. |

_See [Line protocol](/v2.0/reference/syntax/line-protocol/) and [Annotated CSV](/v2.0/reference/syntax/annotated-csv)_

#### Example influx write commands

##### Write a single line of line protocol
```sh
influx write \
  -b bucketName \
  -o orgName \
  -p s \
  'myMeasurement,host=myHost testField="testData" 1556896326'
```

##### Write line protocol from a file
```sh
influx write \
  -b bucketName \
  -o orgName \
  -p s \
  --format=lp
  -f /path/to/line-protocol.txt
```

##### Write annotated CSV from a file
```sh
influx write \
  -b bucketName \
  -o orgName \
  -p s \
  --format=csv
  -f /path/to/data.csv
```