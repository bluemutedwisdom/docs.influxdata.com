---
title: Storage Engine

menu:
  influxdb_1_0:
    weight: 90
    parent: concepts
---

The 0.9 line of InfluxDB used BoltDB as the underlying storage engine.
This writeup is about the Time Structured Merge Tree storage engine that was released in 0.9.5.
There may be small discrepancies between the current implementation of TSM and this document.

<a href="https://influxdata.com/blog/new-storage-engine-time-structured-merge-tree/" target="_">See the blog post announcement about the storage engine here</a>.

## The new InfluxDB storage engine: from LSM Tree to B+Tree and back again to create the Time Structured Merge Tree

The properties of the time series data use case make it challenging for many existing storage engines.
Over the course of InfluxDB’s development we’ve tried a few of the more popular options.
We started with LevelDB, an engine based on LSM Trees, which are optimized for write throughput.
After that we tried BoltDB, an engine based on a memory mapped B+Tree, which is optimized for reads.
Finally, we ended up building our own storage engine that is similar in many ways to LSM Trees.

With our new storage engine we were able to achieve up to a 45x reduction in disk space usage from our B+Tree setup with even greater write throughput and compression than what we saw with LevelDB and its variants.
Using batched writes, we were able to insert more than 300,000 points per second on a c3.8xlarge instance in AWS.
This post will cover the details of that evolution and end with an in-depth look at our new storage engine and its inner workings.

### Properties of Time Series Data

The workload of time series data is quite different from normal database workloads.
There are a number of factors that conspire to make it very difficult to get it to scale and perform well:

* Billions of individual data points
* High write throughput
* High read throughput
* Large deletes to free up disk space
* Mostly an insert/append workload, very few updates

The first and most obvious problem is one of scale.
In DevOps, for instance, you can collect hundreds of millions or billions of unique data points every day.

For example, let’s say we have 200 VMs or servers running, with each server collecting an average of 100 measurements every 10 seconds.
Given there are 86,400 seconds in a day, a single measurement will generate 8,640 points in a day, per server.
That gives us a total of 200 * 100 * 8,640 = 172,800,000 individual data points per day.
We find similar or larger numbers in sensor data use cases.

The volume of data means that the write throughput can be very high.
We regularly get requests for setups than can handle hundreds of thousands of writes per second.
Some larger companies will only consider systems that can handle millions of writes per second.

At the same time, time series data can be a high read throughput use case.
It’s true that if you’re tracking 700,000 unique metrics or time series, you can’t hope to visualize all of them, which is what leads many people to think that you don’t actually read most of the data that goes into the database.
However, other than dashboards that people have up on their screens, there are automated systems for monitoring or combining the large volume of time series data with other types of data.

Inside InfluxDB, we have aggregates that can get calculated on the fly that can combine tens of thousands of time series into a single view.
Each one of those queries does a read on each data point, which means that for InfluxDB, the read throughput is often many times higher than the write throughput.

Given that time series is mostly an append only workload, you might think that it’s possible to get great performance on a B+Tree.
Appends in the keyspace are efficient and you can achieve greater than 100,000 per second.
However, we have those appends happening in individual time series.
So the inserts end up looking more like random inserts than append only inserts.

One of the biggest problems we found with time series data is that it’s very common to delete all data after it gets past a certain age.
The common pattern here is that you’ll have high precision data that is kept for a short period of time like a few hours or a few weeks.
You’ll then downsample and aggregate that data into lower precision, which you’ll keep around for much longer.

The naive implementation would be to simply delete each record once it gets past its expiration time.
However, that means that once you’re up to your window of retention, you’ll be doing just as many deletes as you do writes, which is something most storage engines aren’t designed for.

Let’s dig into the details of the two types of storage engines we tried and how these properties had a significant impact on our performance.

### LevelDB and Log Structured Merge Trees

When the InfluxDB project began, we picked LevelDB as the storage engine because it was what we used for time series data storage for the product that was the precursor to InfluxDB.
We knew that it had great properties for write throughput and everything seemed to just work.

LevelDB is an implementation of a Log Structured Merge Tree (or LSM Tree) that was built as an open source project at Google.
It exposes an API for a key/value store where the key space is sorted.
This last part is important for time series data as it would allow us to quickly go through ranges of time as long as the timestamp was in the key.

LSM Trees are based on a log that takes writes and two structures known as Mem Tables and SSTables.
These tables represent the sorted keyspace.
SSTables are read only files that continuously get replaced by other SSTables that merge inserts and updates into the keyspace.

The two biggest advantages that LevelDB had for us were high write throughput and built in compression.
However, as we learned more about what people needed with time series data, we encountered a few insurmountable challenges.

The first problem we had was that LevelDB doesn’t support hot backups.
If you want to do a safe backup of the database, you have to close it and then copy it.
The LevelDB variants RocksDB and HyperLevelDB fix this problem so we could have moved to them, but there was another problem that was more pressing that we didn’t think could be solved with either of them.

We needed to give our users a way to automatically manage the data retention of their time series data.
That meant that we’d have to do very large scale deletes.
In LSM Trees a delete is as expensive, if not more so, than a write.
A delete will write a new record known as a tombstone.
After that queries will merge the result set with any tombstones to clear out the deletes.
Later, a compaction will run that will remove the tombstone and the underlying record from the SSTable file.

To get around doing deletes, we split data across what we call shards, which are contiguous blocks of time.
Shards would typically hold either a day or 7 days worth of data.
Each shard mapped to an underlying LevelDB.
This meant that we could drop an entire day of data by just closing out the database and removing the underlying files.

Users of RocksDB may at this point bring up a feature called ColumnFamilies.
When putting time series data into Rocks, it’s common to split blocks of time into column families and then drop those when their time is up.
It’s the same general idea that you create a separate area where you can just drop files instead of updating any indexes when you delete a large block of old data.
Dropping a column family is a very efficient operation.
However, column families are a fairly new feature and we had another use case for shards.

Organizing data into shards meant that it could be moved within a cluster without having to examine billions of keys.
At the time of this writing, it was not possible to move a column family in one RocksDB to another.
Old shards are typically cold for writes so moving them around would be cheap and easy and we would have the added benefit of having a spot in the keyspace that is cold for writes so it would be easier to do consistency checks later.

The organization of data into shards worked great for a little while until a large amount of data went into InfluxDB.
For users that had 6 months or a year of data in large databases, they would run out of file handles.
LevelDB splits the data out over many small files.
Having dozens or hundreds of these databases open in a single process ended up creating a big problem.
It’s not something we found with a large number of users, but for anyone that was stressing the database to its limits, they were hitting this problem and we had no fix for it.
There were simply too many file handles open.

### BoltDB and mmap B+Trees

After struggling with LevelDB and its variants for a year we decided to move over to BoltDB, a pure Golang database heavily inspired by LMDB, a mmap B+Tree database written in C.
It has the same API semantics as LevelDB: a key value store where the keyspace is ordered.
Many of our users were surprised about this move after we posted some early testing results of the LevelDB variants vs.
LMDB (a mmap B+Tree) that showed RocksDB as the best performer.

However, there were other considerations that went into this decision outside of the pure write performance.
At this point our most important goal was to get to something stable that could be run in production and backed up.
BoltDB also had the advantage of being written in pure Go, which simplified our build chain immensely and made it easy to build for other OSes and platforms like Windows and ARM.

The biggest win for us was that BoltDB used a single file as the database.
At this point our most common source of bug reports were from people running out of file handles.
Bolt solved the hot backup problem, made it easy to move a shard from one server to another, and the file limit problems all at the same time.

We were willing to take a hit on write throughput if it meant that we’d have a system that was more reliable and stable that we could build on.
Our reasoning was that for anyone pushing really big write loads, they’d be running a cluster anyway.

We released versions 0.9.0 to 0.9.2 based on BoltDB.
From a development perspective it was delightful.
Clean API, fast and easy to build in our Go project, and reliable.
However, after running for a while we found a big problem with write throughput falling over.
After the database got to a certain size (over a few GB) writes would start spiking IOPS.

Some users were able to get past this by putting it on big hardware with near unlimited IOPS.
However, most users are on VMs with limited resources in the cloud.
We had to figure out a way to reduce the impact of writing a bunch of points into hundreds of thousands of series at a time.

With the 0.9.3 and 0.9.4 releases our plan was to put a write ahead log (WAL) in front of Bolt.
That way we could reduce the number of random insertions into the keyspace.
Instead, we’d buffer up multiple writes at once that were next to each other and then flush them.
However, that only served to delay the problem.
High IOPS still became an issue and it showed up very quickly for anyone operating at even moderate work loads.

However, our experience building the first WAL implementation in front of Bolt gave us the confidence we needed that the write problem could be solved.
The performance of the WAL itself was fantastic, the index simply could not keep up.
At this point we started thinking again about how we could create something similar to an LSM Tree that could keep up with our write load.

### The new InfluxDB storage engine and LSM refined

The new InfluxDB storage engine looks very similar to a LSM Tree.
It has a write ahead log and a collection of read-only data files which are  similar in concept to SSTables in an LSM Tree.
These data files, TSM files, contain sorted, compressed series data.

InfluxDB will create a shard for each block of time.
For example, if you have a retention policy with an unlimited duration, shards will get created for each 7 day block of time.
Each of these shards maps to an underlying storage engine database.
Each of these databases has its own WAL and TSM files.

We’ll dig into each of these parts of the storage engine.


#### Storage Engine

The storage engine is ties a number components together as well as provides the external interface used for storing and querying series data.  It is composed of a number of components that each serve a particular role:

* In-Memory Index - The in-memory index is a shared index across shards that provides the quick access to measurements, tags and series.  The index is used by the engine, but is not specific to the storage engine itself.
* WAL - The WAL is a write, optimized storage format that allows for writes to be durable, but not easily queryable.  Writes to the WAL are appended to  segments of a fixed size.
* Cache - The Cache is an in-memory representation of the data stored in the WAL.  It is queried at runtime and merged with the data stored in TSM files.
* TSM Files - TSM files store compressed series data in a columnar format.
* FileStore - The FileStore mediates access to all TSM files on disk.  It ensures that TSM files are installed atomially when existing ones are replaced as well as removing TSM files that are no longer used.
* Compactor - The Compactor is responsible for converting less optimized Cache and TSM data into more read-optimized formats.  It does this by compressing series, removing deleted data, optimizing indes and combining smaller files into larger ones.
* Compaction Planner - The Compaction Planner determines which TSM files are ready for a compaction and ensures that multiple, concurrent compactions do not interfere with each other.
* Compression - Compression is handled by various Encoders and Decoders for a specific data types.  Some encoders are faily static and always encode the same type the same way, other switch their compression strategy based on the shape of the data.
* Writers/Readers - Each file type (WAL segment, TSM files, tombstones, etc..) has writers and reader for working with the formats.


#### Write Ahead Log (WAL)

The WAL is organized as a bunch of files that look like `_000001.wal`.
The file numbers are monotonically increasing and referred to as WAL segments.
When a segment reaches 10MB in size, it is closed and a new one is opened.  Each WAL segment stores multiple compressed blocks of writes and deletes.

When a write comes in with points and optionally new series and fields defined, they are serialized, compressed using Snappy, and written to a WAL file.
The file is fsync’d and the data added to an in-memory index before a success is returned.
This means that batching points together will be required to achieve high throughput performance.

Each entry in the WAL follows a TLV standard with a single byte representing the type of entry (write or delete), a 4 byte `uint32` for the length of the compressed block, and then the compressed block.

#### Cache

The Cache is an in-memory copy of all data points current stored in the WAL.
The points are organized by the key, which is the measurement, tagset, and unique field.
Each field is kept as its own time ordered range.
The data isn’t compressed while in memory.

Queries to the storage engine will merge data from the Cache with data from the TSM files.
When a query runs a copy of the data is made from the cache to be processed by the query engine.
This way writes that come in while a query is happening won’t change the result.

Deletes sent to the Cache will clear out given key or time range for the key.

The Cache exposes a few controls for snapshotting behavior.
The two most important controls are the memory limits.
There is a lower bound, which will trigger a snapshot to TSM files and remove the corresponding WAL segments.
There is also an upper bound limit at which the Cache will start rejecting writes.
This is useful to prevent out of memory situations and we need to apply back pressure to clients writing data.  The checks for memory thresholds occur on every write.

The other snapshot controls are time based.
The idle threshold will have the Cache snapshot to TSM files if it hasn’t received a write within a given amount of time.

Finally, the Cache is reloade on startup by re-reading the WAL from disk.

#### TSM Files

TSM files are a collection of read-only files that are memory mapped.
The structure of these files looks very similar to an SSTable in LevelDB or other LSM Tree variants.

A TSM file is composed for four sections: header, blocks, index and the footer.

```
┌────────┬────────────────────────────────────┬─────────────┬──────────────┐
│ Header │               Blocks               │    Index    │    Footer    │
│5 bytes │              N bytes               │   N bytes   │   4 bytes    │
└────────┴────────────────────────────────────┴─────────────┴──────────────┘
```

Header is composed of a magic number to identify the file type and a version
number.

```
┌───────────────────┐
│      Header       │
├─────────┬─────────┤
│  Magic  │ Version │
│ 4 bytes │ 1 byte  │
└─────────┴─────────┘
```

Blocks are sequences of pairs of CRC32 checksums and data.  The block data is opaque to the
file.  The CRC32 is used for block level error detection.  The length of the blocks
is stored in the index.

```
┌───────────────────────────────────────────────────────────┐
│                          Blocks                           │
├───────────────────┬───────────────────┬───────────────────┤
│      Block 1      │      Block 2      │      Block N      │
├─────────┬─────────┼─────────┬─────────┼─────────┬─────────┤
│  CRC    │  Data   │  CRC    │  Data   │  CRC    │  Data   │
│ 4 bytes │ N bytes │ 4 bytes │ N bytes │ 4 bytes │ N bytes │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

Following the blocks is the index for the blocks in the file.
The index is
composed of a sequence of index entries ordered lexicographically by key and
then by time.
Each index entry starts with a key length and key followed by the block type (float, int, bool, string) and a count of the number of blocks in the file.  Each block entry is composed of the min and max time for the block, the offset into the file where the block is located and the the size of the block.

The index structure can provide efficient access to all blocks as well as the
ability to determine the cost associated with accessing a given key.
Given a key and timestamp, we can determine whether a file contains the block for that
timestamp as well as where that block resides and how much data to read to
retrieve the block.
If we know we need to read all or multiple blocks in a
file, we can use the size to determine how much to read in a given IO.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                   Index                                    │
├─────────┬─────────┬──────┬───────┬─────────┬─────────┬────────┬────────┬───┤
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
└─────────┴─────────┴──────┴───────┴─────────┴─────────┴────────┴────────┴───┘
```

The last section is the footer that stores the offset of the start of the index.

```
┌─────────┐
│ Footer  │
├─────────┤
│Index Ofs│
│ 8 bytes │
└─────────┘
```

#### Compression

Each block is compressed to reduce storage space and disk IO when querying.
A block contains the timestamps and values for a given series and field.
Each block has one byte header, followed by the compressed timestamps and then the compressed values.

```
┌───────┬─────┬─────────────────┬──────────────────┐
│ Type  │ Len │   Timestamps    │      Values      │
│1 Byte │VByte│     N Bytes     │     N Bytes      │
└───────┴─────┴─────────────────┴──────────────────┘
```

The timestamps and values are stored separately as two compressed parts using different encodings depending on the data type and its shape.
Storing them independently allows timestamp encoding to be used with different field types more easily.
It also allows for different encodings to be used depending on the type of the data.
For example, some points may be able to use run-length encoding whereas other may not.

Each value type also contains a 1 byte header indicating the type of compression for the remaining bytes.
The four high bits store the compression type and the four low bits are used by the encoder if needed.

##### Timestamps

Timestamp encoding is adaptive and based on the structure of the timestamps that are encoded.
It uses a combination of delta encoding, scaling and compression using simple8b, run-length encoding as well as falling back to no compression if needed.

Timestamp resolution can be as granular as a nanosecond and, uncompressed, require 8 bytes to store.
During encoding, the values are first delta-encoded.
The first value is the starting timestamp and subsequent values are the differences from the prior value.
This usually converts the values into much smaller integers that are easier to compress.
Many timestamps are also monotonically increasing and fall on even boundaries of time such as every 10s.
When timestamps have this structure, they are scaled by the largest common divisor that is also a factor of 10.
This has the effect of converting very large integer deltas into smaller ones that compress even better.

Using these adjusted values, if all the deltas are the same, the time range is stored using run-length encoding.
If run-length encoding is not possible and all values are less than 1 << 60 - 1 (~36.5 yrs in nanosecond resolution), then the timestamps are encoded using simple8b encoding which is a 64bit word-aligned integer encoding.
This encoding packs up multiple integers into a single 64bit word.
If any value exceeds the maximum values, the deltas are stored uncompressed using 8 bytes each for the block.
Future encodings may use a patched scheme such as Patched Frame-Of-Reference (PFOR) to handle outliers more effectively.

##### Floats

Floats are encoded using an implementation of the Facebook Gorilla paper.
This encoding XORs consecutive values together which produces a small result when the values are close together.
The delta is then stored using control bits to indicate how many leading and trailing zero are in the XOR value.
Our version removes the timestamp encoding, as described in paper, and only encodes the float values.

##### Integers

Integer encoding uses two different strategies depending on the range of values in the uncompressed data.
Encoded values are first encoded using zig zag encoding which is also used for signed integers in Google Protocol Buffers.
This interleaves positive and negative integers across a range of positive integers.

For example, [-2,-1,0,1] becomes [3,1,0,2].
See https://developers.google.com/protocol-buffers/docs/encoding?hl=en#signed-integers for more information.

If all the zig zag encoded values less than 1 << 60 - 1, they are compressed using the simple8b encoding.
If any values is larger than the maximum value, then values are stored uncompressed in the block.
If all of the values are the same, run-length encoding is used.  This works very well for values are 0 or some constant value almost always.

##### Booleans

Booleans are encoded using a simple bit packing strategy where each boolean uses 1 bit.
The number of booleans encoded is stored using variable-byte encoding at the beginning of the block.

##### Strings
Strings are encoding using Snappy compression.
Each string is packed next each other in order and compressed as one larger block.

#### Compactions

Compactions are a peridoc process that migrates data stored in write optimized format into a more read optimized format.  The are a number of compactions that take place while a shard is hot for writes:

* Snapshots - Values in the Cache and WAL need to be converted to TSM files to free memory and disk space used by the WAL segments.  These occur based on the cache memory and time thresholds.
* Level Compactions - Level compactions (1-4) occur as the shape of data on disk in TSM files grows.  TSM files are compacted from snapshots to level 1 files.  Multiple level 1 files are compacted to produce level 2 files.  The process continues until files level 4 and reach a max size where they remain until deletes or later compactions optimized them further.  Lower level compactions are use compactions strategies that avoid decompressing and combining blocks which is CPU intensive.  Higher levels will re-combine blocks to full pack them and increase the compression ratio.
* Index Optimization - When many level 4 TSM files accumulate, the internal indexes start to become larger and more costly to access.  An index optimization compaction will run to split the series and indexes across a new set of TSM files.
* Full Compactions - Full compactions run when a shard has become cold for writes for long time or deletes have occurred on the shard.  Full compactions produce an optimal set of TSM files.  Once a shard is fully compacted, no other compactions will run on it unless new writes or deletes are stored.


#### Writes

Writes are appended to the current WAL segment and are also added to the Cache.
Each WAL segment is size bounded and rolls-over to a new file after it fills up.
The cache is also size bounded; snapshots are taken and WAL compactions are initiated when the cache becomes too full.
If the inbound write rate exceeds the WAL compaction rate for a sustained period, the cache may become too full in which case new writes will fail until the snapshot process catches up.

When WAL segments fill up and have been closed, the Compactor snapshosts the Cache and writes the data to a new TSM file.
When the TSM file is successfully written and fsync'd, it is loaded and referenced by the FileStore.

#### Updates

Updates (writing a newer value for a point that already exists) occur as normal writes.
Since cached values overwrite existing values, newer writes take precedence.
If a write is would overwrite a point in a prior TSM file, the points are merged at runtime and the newer write takes precedenc.


#### Deletes

Deletes occur by writing a delete entry for the measurement or series to the WAL and then updating the Cache and FileStore.
The Cache evicts all relevant entries.
The FileStore writes a tombstone file for each TSM file that contains relevant data.
These tombstone files are used at startup time to ignore blocks as well as during compactions to remove deleted entries.

Queries agains partially deleted series are handled at query time until a compaction removes the data fully from the TSM files.

#### Queries

When a query is executed by the storage engine, it is a seek to a given time associated with a specific series key and field.
First, we do a search on the data files to find the files that have a time range that matches the time we’re seeking to as well as contains the series we're searching.

Once we have the data files, we need to find the position in the file for the series keys index entries.
We run a binary search against each TSM index to find the location of it's index blocks.

In the common cases, the blocks will not overlap across multiple TSM files and we can linearly search the index entries to find the start block to read from.
If there are overlapping blocks of time, the indexe entries are sorted to ensure newer writes will overwrite older ones and that blocks can be iterated over in order during query execution.

When iterating over the index entries, the blocks are read sequentially from the blocks section.
The blocks is decompressed and we seek to that specific point.