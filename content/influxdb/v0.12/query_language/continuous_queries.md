---
title: Continuous Queries

menu:
  influxdb_012:
    weight: 40
    parent: query_language
---

When writing large amounts of data to InfluxDB, you may often want to downsample the raw data, that is, use `GROUP BY time()` with an InfluxQL [function](/influxdb/v0.12/query_language/functions/) to change the high frequency data into lower frequency data. Repeatedly running the same queries by hand can be tedious. InfluxDB's continuous queries (CQ) simplify the downsampling process; CQs run automatically and write the query results to another measurement.

* [CQ definition](/influxdb/v0.12/query_language/continuous_queries/#cq-definition)
* [InfluxQL for creating a CQ](/influxdb/v0.12/query_language/continuous_queries/#influxql-for-creating-a-cq)  
&nbsp;&nbsp;&nbsp;◦&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[The `CREATE CONTINUOUS QUERY` statement](/influxdb/v0.12/query_language/continuous_queries/#the-create-continuous-query-statement)  
&nbsp;&nbsp;&nbsp;◦&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[CQs with backreferencing](/influxdb/v0.12/query_language/continuous_queries/#cqs-with-backreferencing)  
* [List CQs with `SHOW`](/influxdb/v0.12/query_language/continuous_queries/#list-cqs-with-show)
* [Delete CQs with `DROP`](/influxdb/v0.12/query_language/continuous_queries/#delete-cqs-with-drop)
* [Backfilling](/influxdb/v0.12/query_language/continuous_queries/#backfilling)
* [Further reading](/influxdb/v0.12/query_language/continuous_queries/#further-reading)

## CQ definition
A CQ is an InfluxQL query that the system runs automatically and periodically within a database. InfluxDB stores the results of the CQ in a specified [measurement](/influxdb/v0.12/concepts/glossary/#measurement). CQs require a function in the `SELECT` clause and must include a `GROUP BY time()` clause.

CQs do not maintain any state. Each execution of a CQ is a standalone query that resamples all points in the database matching the conditions of the query.

The time ranges of the CQ results have round-number boundaries that are set internally by the database. There is currently no way for users to alter the start or end times of the intervals.

Only admin users are allowed to work with continuous queries. For more on user privileges, see [Authentication and Authorization](/influxdb/v0.12/administration/authentication_and_authorization/#user-types-and-their-privileges).

> **Note:** CQs only execute on data received after the CQ's creation. If you'd like to downsample data written to InfluxDB before the CQ was created, see the examples in [Data Exploration](/influxdb/v0.12/query_language/data_exploration/#downsample-data).

## InfluxQL for creating a CQ
### The `CREATE CONTINUOUS QUERY` statement
```sql
CREATE CONTINUOUS QUERY <cq_name> ON <database_name> [RESAMPLE [EVERY <interval>] [FOR <interval>]] BEGIN SELECT <function>(<stuff>)[,<function>(<stuff>)] INTO <different_measurement> FROM <current_measurement> [WHERE <stuff>] GROUP BY time(<interval>)[,<stuff>] END
```

The `CREATE CONTINUOUS QUERY` statement is essentially an InfluxQL query surrounded by `CREATE CONTINUOUS QUERY [...] BEGIN` and `END`.
The following discussion breaks the CQ statement into its meta portion (everything between `CREATE` and `BEGIN`) and query portion (everything between `BEGIN` and `END`).

A successful `CREATE CONTINUOUS QUERY` statement returns an empty response.
If you attempt to create a continuous query that already exists, InfluxDB does
not return an error.

#### Meta syntax:
```
CREATE CONTINUOUS QUERY ON <database_name> [RESAMPLE [EVERY <interval>] [FOR <interval>]]
```

A CQ belongs to a database.
Specify the database where you want the CQ to live with `ON <database_name>`.  

The optional `RESAMPLE` clause determines how often InfluxDB runs the CQ (`EVERY <interval>`) and the time range over which InfluxDB runs the CQ (`FOR <interval>`).
If included, the `RESAMPLE` clause must specify either `EVERY`, or `FOR`, or both.
Without the `RESAMPLE` clause, InfluxDB runs the CQ at the same interval as the `GROUP BY time()` interval and it calculates the query for the most recent `GROUP BY time()` interval (that is, where time is between `now()` and `now()` minus the `GROUP BY time()` interval).

#### Query syntax:
```
BEGIN SELECT <function>(<stuff>)[,<function>(<stuff>)] INTO <different_measurement> FROM <current_measurement> [WHERE <stuff>] GROUP BY time(<interval>)[,<stuff>] END
```

The query portion of the statement differs from a typical `SELECT [...] GROUP BY (time)` statement in two ways:

1. The `INTO` clause:
This is where you specify the destination measurement for the query results.

2. The optional `WHERE` clause:
Because CQs run on regularly incremented time intervals you don't need to (and shouldn't!) specify a time range in the `WHERE` clause. When included, the CQ's `WHERE` clause should filter information about tags.

> **Note:** If you include a tag in the CQ's `SELECT` clause, InfluxDB changes the tag in `<current_measurement>` to a field in `<different_measurement>`.
To preserve a tag in `<different_measurement>`, only include the tag key in the CQ's `GROUP BY` clause.

> If you specify a tag in the CQ's `SELECT` clause **and** in the CQ's `GROUP BY` clause, you will not be able to query the data in `<different_measurement>`.
See GitHub Issue [#4630](https://github.com/influxdata/influxdb/issues/4630) for more information.

#### CQ examples:

* Create a CQ with one function:

    ```sql
> CREATE CONTINUOUS QUERY minnie ON world BEGIN SELECT min(mouse) INTO min_mouse FROM zoo GROUP BY time(30m) END
    ```

    Once executed, InfluxDB automatically calculates the 30 minute minimum of the field `mouse` in the measurement `zoo`, and it writes the results to the measurement `min_mouse`.
    Note that the CQ `minnie` only exists in the database `world`.

* Create a CQ with one function and write the results to another [retention policy](/influxdb/v0.12/concepts/glossary/#retention-policy-rp):

    ```sql
> CREATE CONTINUOUS QUERY minnie_jr ON world BEGIN SELECT min(mouse) INTO world."7days".min_mouse FROM world."1day".zoo GROUP BY time(30m) END
    ```

    The CQ `minnie_jr` acts in the same way as the CQ `minnie`, however, InfluxDB calculates the 30 minute minimum of the field `mouse` in the measurement `zoo` and under the retention policy `1day`, and it automatically writes the results of the query to the measurement `min_mouse` under the retention policy `7days`.

    Combining CQs and retention policies provides a useful way to automatically downsample data and expire the unnecessary raw data.
    For a complete discussion on this topic, see [Downsampling and Data Retention](/influxdb/v0.12/guides/downsampling_and_retention/).

* Create a CQ with two functions:

    ```sql
> CREATE CONTINUOUS QUERY minnie_maximus ON world BEGIN SELECT min(mouse),max(imus) INTO min_max_mouse FROM zoo GROUP BY time(30m) END
    ```

    The CQ `minnie_maximus` automatically calculates the 30 minute minimum of the field `mouse` and the 30 minute maximum of the field `imus` (both fields are in the measurement `zoo`), and it writes the results to the measurement `min_max_mouse`.

* Create a CQ with two functions and personalize the [field keys](/influxdb/v0.12/concepts/glossary/#field-key) in the results:

    ```sql
> CREATE CONTINUOUS QUERY minnie_maximus_1 ON world BEGIN SELECT min(mouse) AS minuscule,max(imus) AS monstrous INTO min_max_mouse FROM zoo GROUP BY time(30m) END
    ```

    The CQ `minnie_maximus_1` acts in the same way as `minnie_maximus`, however, InfluxDB names field keys `miniscule` and `monstrous` in the destination measurement instead of `min` and `max`.
    For more on `AS`, see [Functions](/influxdb/v0.12/query_language/functions/#rename-the-output-column-s-title-with-as).

* Create a CQ with a 30 minute `GROUP BY time()` interval that runs every 15 minutes:

    ```sql
> CREATE CONTINUOUS QUERY vampires ON transylvania RESAMPLE EVERY 15m BEGIN SELECT count(dracula) INTO vampire_populations FROM raw_vampires GROUP BY time(30m) END
    ```

    Without `RESAMPLE EVERY 15m`, `vampires` would run every 30 minutes - the same interval as the `GROUP BY time()` interval.

* Create a CQ with a 30 minute `GROUP BY time()` interval that runs every 30 minutes and computes the query for all `GROUP BY time()` intervals within the last hour:

    ```sql
> CREATE CONTINUOUS QUERY vampires_1 ON transylvania RESAMPLE FOR 60m BEGIN SELECT count(dracula) INTO vampire_populations_1 FROM raw_vampires GROUP BY time(30m) END
    ```

    InfluxDB runs `vampires_1` every 30 minutes (the same interval as the `GROUP BY time()` interval) and it computes two queries per run:
    one where time is between `now()` and `now() - 30m` and one where time is between `now() - 30m` and `now() - 60m`.
    Without the `RESAMPLE` clause, InfluxDB would compute the query for only one 30 minute interval, that is, where time is between `now()` and `now() - 30m`.

* Create a CQ with a 30 minute `GROUP BY time()` interval that runs every 15 minutes and computes the query for all `GROUP BY time()` intervals within the last hour:

    ```sql
> CREATE CONTINUOUS QUERY vampires_2 ON transylvania RESAMPLE EVERY 15m FOR 60m BEGIN SELECT count(dracula) INTO vampire_populations_2 FROM raw_vampires GROUP BY time(30m) END
    ```

    `vampires_2` runs every 15 minutes and computes two queries per run:
    one where time is between `now()` and `now() - 30m` and one where time is between `now() - 30m` and `now() - 60m`

### CQs with backreferencing
Use `:MEASUREMENT` in the `INTO` statement to backreference measurement names:
```sql
CREATE CONTINUOUS QUERY <cq_name> ON <database_name> BEGIN SELECT <function>(<stuff>)[,<function>(<stuff>)] INTO <database_name>.<retention_policy>.:MEASUREMENT FROM </relevant_measurement(s)/> [WHERE <stuff>] GROUP BY time(<interval>)[,<stuff>] END
```

*CQ backreferencing example:*
<br>
<br>
```sql
> CREATE CONTINUOUS QUERY elsewhere ON fantasy BEGIN SELECT mean(value) INTO reality."default".:MEASUREMENT FROM /elf/ GROUP BY time(10m) END
```
The CQ `elsewhere` automatically calculates the 10 minute average of the field `value` in each `elf` measurement in the database `fantasy`. It writes the results to the already-existing database `reality`, preserving all of the measurement names in `fantasy`.

A sample of the data in `fantasy`:
```bash
> SHOW MEASUREMENTS
name: measurements
------------------
name
elf1
elf2
wizard
>
> SELECT * FROM elf1
name: cpu_usage_idle
--------------------
time			               value
2015-12-19T01:15:30Z	 97.76333874796951
2015-12-19T01:15:40Z	 98.3129217695576
[...]
2015-12-19T01:36:00Z	 94.71778221778222
2015-12-19T01:35:50Z	 87.8
```

A sample of the data in `reality` after `elsewhere` runs for a bit:
```bash
> SHOW MEASUREMENTS
name: measurements
------------------
name
elf1
elf2
>
> SELECT * FROM elf1
name: elf1
--------------------
time			               mean
2015-12-19T01:10:00Z	 97.11668879244841
2015-12-19T01:20:00Z	 94.50035091670394
2015-12-19T01:30:00Z	 95.99739053789172
```

## List CQs with `SHOW`
List every CQ by database with:
```sql
SHOW CONTINUOUS QUERIES
```

*Example:*

```bash
> SHOW CONTINUOUS QUERIES
name: reality
-------------
name	query

name: fantasy
-------------
name		     query
elsewhere	 CREATE CONTINUOUS QUERY elsewhere ON fantasy BEGIN SELECT mean(value) INTO reality."default".:MEASUREMENT FROM fantasy."default"./cpu/ WHERE cpu = 'cpu-total' GROUP BY time(10m) END
```

The output shows that the database `reality` has no CQs and the database `fantasy` has one CQ called `elsewhere`.

## Delete CQs with `DROP`
Delete a CQ from a specific database with:
```sql
DROP CONTINUOUS QUERY <cq_name> ON <database_name>
```

*Example:*

```bash
> DROP CONTINUOUS QUERY elsewhere ON fantasy
>
```

A successful `DROP CONTINUOUS QUERY` returns an empty response.

## Backfilling

CQs do not backfill data, that is, they do not compute results for data written to the database before the CQ existed. Instead, users can backfill data with the `INTO` clause. Unlike CQs, backfill queries require a `WHERE` clause with a `time` restriction.

### Examples

Here is a basic backfill example:
```sql
> SELECT min(temp) as min_temp, max(temp) as max_temp INTO "reading.minmax.5m" FROM reading
WHERE time >= '2015-12-14 00:05:20' AND time < '2015-12-15 00:05:20'
GROUP BY time(5m)
```

Tags (`sensor_id` in the example below) can be used optionally in the same way as in CQs:
```sql
> SELECT min(temp) as min_temp, max(temp) as max_temp INTO "reading.minmax.5m" FROM reading
WHERE time >= '2015-12-14 00:05:20' AND time < '2015-12-15 00:05:20'
GROUP BY time(5m), sensor_id
```

To prevent the backfill from creating a huge number of "empty" points containing only `null` values, [fill()](/influxdb/v0.12/query_language/data_exploration/#the-group-by-clause-and-fill) can be used at the end of the query:
```sql
> SELECT min(temp) as min_temp, max(temp) as max_temp INTO "reading.minmax.5m" FROM reading
WHERE time >= '2015-12-14 00:05:20' AND time < '2015-12-15 00:05:20'
GROUP BY time(5m), fill(none)
```

If you would like to further break down the queries and run them with even more control, you can add additional `WHERE` clauses:
```sql
> SELECT min(temp) as min_temp, max(temp) as max_temp INTO "reading.minmax.5m" FROM reading
WHERE sensor_id="EG-21442" AND time >= '2015-12-14 00:05:20' AND time < '2015-12-15 00:05:20'
GROUP BY time(5m)
```

## Further reading
Now that you know how to create CQs with InfluxDB, check out [Downsampling and Data Retention](/influxdb/v0.12/guides/downsampling_and_retention/) for how to combine CQs with retention policies to automatically downsample data and expire unnecessary data.
