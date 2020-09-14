---
title: Upgrade to InfluxDB OSS 2.0rc0
description: >
  Upgrade to InfluxDB OSS 2.0rc0.
menu:
  influxdb_2_0_ref:
    name: Upgrade to InfluxDB OSS 2.0rc0
weight: 9
---

To upgrade from InfluxDB 2.0 beta 16 or earlier to InfluxDB 2.0rc0, you must manually upgrade all resources and data to the latest version by completing the following steps:

- [1. Disable existing integrations](#)
- [2. Stop existing InfluxDB beta instance](#)
- [3. (Optional) Rename existing InfluxDB binaries](#)
- [4. Move existing data and start the latest InfluxDB](#)
- [5. Start old InfluxDB beta instance](#)
- [6. Create configuration profiles for the InfluxDB CLI](#)
- [7. Copy all resources from old instance to the new one](#)
- [8. Setup integrations to point to new instance](#)
- [9. Load historical data into new instance](#)

Depending on how you have things set up and how important the data stored in InfluxDB is,
you can choose the parts of this guide that make sense to you.
For example, if there's nothing in your existing InfluxDB beta instance you need to keep, feel free to skip the sections about migrating resources and data to the new instance.

### Why is this manual process required?

To ensure that existing InfluxQL integrations work with the latest release,
we had to make breaking changes to the underlying storage engine for InfluxDB 2.0.

{{% note %}}
If you have questions upgrading please
[open an issue](https://github.com/influxdata/influxdb/issues)
or join the [Community Slack workspace](https://influxcommunity.slack.com/) to get immediate help.
{{% /note %}}

## 1. Disable existing integrations

To begin, shut off all integrations that are reading, writing, or monitoring your InfluxDB instance.
This includes Telegraf, client libraries, and any custom applications.

## 2. Stop existing InfluxDB beta instance

Next, shut down your existing InfluxDB beta instance.
You can manually kill the individual process using **Control+c**
(or by finding the process ID with `ps aux | grep -i influxd` and using `sudo kill -9 <PID>`).
If you've set `influxd` to run as a system process, follow the same steps you would use to disable any system process.

## 3. (Optional) Rename existing InfluxDB binaries

To easily identify your existing InfluxDB binaries, rename them `influx_old` and `influxd_old`.
This is helpful if you've installed the binaries in your path.
We use the names `influxd_old` for this guide, but you can use whatever you like.

## 4. Move existing data and start the latest InfluxDB

If you have not already, [download the InfluxDB OSS 2.0rc0](https://portal.influxdata.com/downloads/).
Download and follow the install instructions for `influxd` and `influx`.
Be careful not to overwrite the existing binaries(that they may or may not have renamed earlier).

To move data between the two instances, first configure both the old and new instances of InfluxDB to run at the same time.
If you download the latest InfluxDB beta and try to start it with existing data,
most likely it won't start and you'll see the following error message in your terminal:

```
Incompatible InfluxDB 2.0 version found.
Move all files outside of engine_path before influxd will start.
```

To ensure you don't get this message, run the following command to move your existing data to a another location (anywhere you like):

```sh
mv ~/.influxdbv2 ~/.influxdbv2_old
```

(You can move it to wherever you'd like.)
When we start the old instance again, we will tell it where your data files are located.

Start the latest InfluxDB version by running

```sh
influxd
```

{{% note %}}
If you were using specific [command line flags](/influxdb/v2.0/reference/cli/influxd/#flags) for InfluxDB beta, you can use those same command line flags.
{{% /note %}}

Since the data folder has been moved, everything will be empty.
You can visit http://localhost:8086 in your browser and see a setup page, but don't go through the setup process yet.

## 5. Start old InfluxDB beta instance

You can now start your old influxdb instance and point it to your old data directory:

```sh
./influxd_old \
    --bolt-path ~/.influxdbv2_old/influxd.bolt \
    --engine-path ~/.influxdbv2_old/engine
```

Double check that InfluxDB is working by going to your previous InfluxDB beta location (probably http://localhost:9999) and logging in.
Everything should still be there.

{{% warn %}}
Backup your old `influxd.bolt` file before doing this, as manually editing this file could cause you to lose all your data.
{{% /warn %}}

{{% note %}}
If you run into a problem about a missing migration, you can manually edit your bolt file to remove it.
You can use BoltBrowser to open and edit your old influxd.bolt file and manually remove the offending migration.

Highlight the record under the `migrationsv1` path and press **D**.
{{% /note %}}

Now, the new instance and old instance of InfluxDB are running simultaneously.

## 6. Create configuration profiles for the InfluxDB CLI

Next, set up your InfluxDB CLI to connect to your old and new instances.

### Configure old profile

If you've used the CLI before, copy your existing `configs` file to your new data directory:

```sh
cp ~/.influxdbv2_old/configs ~/.influxdbv2/configs
```

We recommend renaming the old configuration file to something like `influx_old`.

{{< keep-url >}}
```toml
[influx_old]
  url = "http://localhost:9999"
  token = "<YOUR TOKEN>"
  org = "influxdata"
  active = true
```

If you've never used the CLI before, create a new configuration profile to connect to your old instance using the `influx config` command.

{{< keep-url >}}
```sh
influx config create \
    --config-name influx_old \
    --host-url http://localhost:9999 \
    --org influxdata \
    --token <OLD_TOKEN>
```

Now when you run `influx config ls`, you will see a profile for your old instance.

{{< keep-url >}}
```sh
$ influx config ls
Active  Name        URL                    Org
*       influx_old  http://localhost:9999  InfluxData
```

### Configure new profile

Next, let's set up your new instance which will automatically create a configuration profile for you.
You can use the same username and password as your old instance, or something completely new, it's up to you.

{{% warn %}}
For a bucket name, make sure you don't use the same name as a bucket in your existing instance, otherwise, there will be a collision when you try to copy your resources over.
We will delete this dummy bucket after everything is moved over.
{{% /warn %}}

```
$ influx setup
Welcome to InfluxDB 2.0!
Please type your primary username: admin

Please type your password:

Please type your password again:

Please type your primary organization name: InfluxData

Please type your primary bucket name: dummy_bucket

Please type your retention period in hours.
Or press ENTER for infinite.:

You have entered:
  Username:          admin
  Organization:      InfluxData
  Bucket:            dummy_bucket
  Retention Period:  infinite
Confirm? (y/n): y

Config default has been stored in /Users/rsavage/.influxdbv2/configs.
User	Organization	Bucket
admin	InfluxData	dummy_bucket
```

You now have two config profiles: one named `default` which points to your new instance, and one named `influx_old` that points to your old instance.

{{< keep-url >}}
```sh
$ influx config ls
Active  Name        URL                    Org
        default     http://localhost:8086  InfluxData
*       influx_old  http://localhost:9999  InfluxData
```

<!-- TODO add docs link below for profiles... -->
<!-- TODO is this flag correct?-->
Now you can send commands to each of them as needed using the [`-c, --active-config`]() option on the CLI.

## 7. Copy all resources from old instance to the new one

Now we can copy all your existing InfluxDB resources, such as dashboards, tasks, and alerts to your new instance using a single command.

```sh
influx export all -c influx_old | influx apply -c default
```

(To learn more about this command, see
[`influx export`](/influxdb/v2.0/reference/cli/influx/export/) and
[`influx apply`](/influxdb/v2.0/reference/cli/influx/apply/).)

This displays a list of the resources being created in your new instance.
Assuming everything succeeded, you can feel free to delete the bucket created during the setup.

```sh
$ influx export all -c influx_old | influx apply -c default
LABELS    +add | -remove | unchanged
+-----+------------------------+----+---------------+---------+-------------+
| +/- |     METADATA NAME      | ID | RESOURCE NAME |  COLOR  | DESCRIPTION |
+-----+------------------------+----+---------------+---------+-------------+
| +   | tasty-northcutt-c9c001 |    | something     | #326BBA |             |
+-----+------------------------+----+---------------+---------+-------------+
|                                                      TOTAL  |      1      |
+-----+------------------------+----+---------------+---------+-------------+
​
BUCKETS    +add | -remove | unchanged
+-----+------------------------+----+---------------+------------------+-------------+
| +/- |     METADATA NAME      | ID | RESOURCE NAME | RETENTION PERIOD | DESCRIPTION |
+-----+------------------------+----+---------------+------------------+-------------+
| +   | fasting-taussig-c9c007 |    | apps          | 0s               |             |
+-----+------------------------+----+---------------+------------------+-------------+
+-----+------------------------+----+---------------+------------------+-------------+
| +   | great-davinci-c9c005   |    | new_telegraf  | 0s               |             |
+-----+------------------------+----+---------------+------------------+-------------+
+-----+------------------------+----+---------------+------------------+-------------+
| +   | stubborn-hugle-c9c003  |    | telegraf      | 719h59m59s       |             |
+-----+------------------------+----+---------------+------------------+-------------+
|                                                          TOTAL       |      3      |
+-----+------------------------+----+---------------+------------------+-------------+
```

{{% note %}}
We are using config profiles here to export all the resources from the old instance and applying them to your new instance.
The only things that will not be copied over are scraper configurations.
You will need to manually reconfigure those.
{{% /note %}}

Now you have all the resources from your old instance stored in your new instance.
Sign in to your new instance (by default http://localhost:8086) and take a look.
You will see dashboards, but your old data has yet been migrated.

## 8. Setup integrations to point to new instance

Now is a good time to set up any integrations you needed to disable at the beginning of this process.
Telegraf, client libraries, custom applications,
or third-party data sinks will need to be setup again using new tokens and credentials.

## 9. Load your historical data into new instance

Use the CLI to export and then re-import your data using the command below.
(For the range, pick a time before your bucket's retention period, or something a really long time ago if you have an unlimited retention period.)

```sh
influx query -c influx_old \
  'from(bucket: "my-bucket") |> range(start: -3y)' --raw > my-bucket.csv
```

Then write to the new bucket:

```sh
influx write -c default --format csv -b my-bucket -f my-bucket.csv
```

Repeat that process for each bucket.

## Verify InfluxDB resources, data, and integrations

To verify that the latest version of InfluxDB is running with all your resources, data, and integrations configured,
Double check and make sure everything is there and it is working as expected.
Once set up with the latest InfluxDB, you can safely turn off your old instance and archive the previous data directory.
