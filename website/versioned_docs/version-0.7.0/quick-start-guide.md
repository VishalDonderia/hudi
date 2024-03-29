---
version: 0.7.0
title: "Quick-Start Guide"
toc: true
last_modified_at: 2019-12-30T15:59:57-04:00
---

This guide provides a quick peek at Hudi's capabilities using spark-shell. Using Spark datasources, we will walk through 
code snippets that allows you to insert and update a Hudi table of default table type: 
[Copy on Write](/docs/concepts#copy-on-write-table). 
After each write operation we will also show how to read the data both snapshot and incrementally.
# Scala example

## Setup

Hudi works with Spark-2.x & Spark 3.x versions. You can follow instructions [here](https://spark.apache.org/downloads) for setting up spark. 
From the extracted directory run spark-shell with Hudi as:

```scala
// spark-shell
spark-shell \
  --packages org.apache.hudi:hudi-spark-bundle_2.12:0.7.0,org.apache.spark:spark-avro_2.12:3.0.1 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
```

<div className="notice--info">
  <h4>Please note the following: </h4>
<ul>
  <li>spark-avro module needs to be specified in --packages as it is not included with spark-shell by default</li>
  <li>spark-avro and spark versions must match (we have used 3.0.1 for both above)</li>
  <li>we have used hudi-spark-bundle built for scala 2.12 since the spark-avro module used also depends on 2.12. 
         If spark-avro_2.11 is used, correspondingly hudi-spark-bundle_2.11 needs to be used. </li>
</ul>
</div>

Setup table name, base path and a data generator to generate records for this guide.

```scala
// spark-shell
import org.apache.hudi.QuickstartUtils._
import scala.collection.JavaConversions._
import org.apache.spark.sql.SaveMode._
import org.apache.hudi.DataSourceReadOptions._
import org.apache.hudi.DataSourceWriteOptions._
import org.apache.hudi.config.HoodieWriteConfig._

val tableName = "hudi_trips_cow"
val basePath = "file:///tmp/hudi_trips_cow"
val dataGen = new DataGenerator
```

The [DataGenerator](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L50) 
can generate sample inserts and updates based on the the sample trip schema [here](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L57)
{: .notice--info}


## Insert data

Generate some new trips, load them into a DataFrame and write the DataFrame into the Hudi table as below.

```scala
// spark-shell
val inserts = convertToStringList(dataGen.generateInserts(10))
val df = spark.read.json(spark.sparkContext.parallelize(inserts, 2))
df.write.format("hudi").
  options(getQuickstartWriteConfigs).
  option(PRECOMBINE_FIELD_OPT_KEY, "ts").
  option(RECORDKEY_FIELD_OPT_KEY, "uuid").
  option(PARTITIONPATH_FIELD_OPT_KEY, "partitionpath").
  option(TABLE_NAME, tableName).
  mode(Overwrite).
  save(basePath)
``` 

`mode(Overwrite)` overwrites and recreates the table if it already exists.
You can check the data generated under `/tmp/hudi_trips_cow/<region>/<country>/<city>/`. We provided a record key 
(`uuid` in [schema](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L58)), partition field (`region/country/city`) and combine logic (`ts` in 
[schema](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L58)) to ensure trip records are unique within each partition. For more info, refer to 
[Modeling data stored in Hudi](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=113709185#FAQ-HowdoImodelthedatastoredinHudi)
and for info on ways to ingest data into Hudi, refer to [Writing Hudi Tables](/docs/writing_data).
Here we are using the default write operation : `upsert`. If you have a workload without updates, you can also issue 
`insert` or `bulk_insert` operations which could be faster. To know more, refer to [Write operations](/docs/writing_data#write-operations)
{: .notice--info}

## Query data 

Load the data files into a DataFrame.

```scala
// spark-shell
val tripsSnapshotDF = spark.
  read.
  format("hudi").
  load(basePath + "/*/*/*/*")
//load(basePath) use "/partitionKey=partitionValue" folder structure for Spark auto partition discovery
tripsSnapshotDF.createOrReplaceTempView("hudi_trips_snapshot")

spark.sql("select fare, begin_lon, begin_lat, ts from  hudi_trips_snapshot where fare > 20.0").show()
spark.sql("select _hoodie_commit_time, _hoodie_record_key, _hoodie_partition_path, rider, driver, fare from  hudi_trips_snapshot").show()
```

This query provides snapshot querying of the ingested data. Since our partition path (`region/country/city`) is 3 levels nested 
from base path we ve used `load(basePath + "/*/*/*/*")`. 
Refer to [Table types and queries](/docs/concepts#table-types--queries) for more info on all table types and query types supported.
{: .notice--info}

## Update data

This is similar to inserting new data. Generate updates to existing trips using the data generator, load into a DataFrame 
and write DataFrame into the hudi table.

```scala
// spark-shell
val updates = convertToStringList(dataGen.generateUpdates(10))
val df = spark.read.json(spark.sparkContext.parallelize(updates, 2))
df.write.format("hudi").
  options(getQuickstartWriteConfigs).
  option(PRECOMBINE_FIELD_OPT_KEY, "ts").
  option(RECORDKEY_FIELD_OPT_KEY, "uuid").
  option(PARTITIONPATH_FIELD_OPT_KEY, "partitionpath").
  option(TABLE_NAME, tableName).
  mode(Append).
  save(basePath)
```

Notice that the save mode is now `Append`. In general, always use append mode unless you are trying to create the table for the first time.
[Querying](#query-data) the data again will now show updated trips. Each write operation generates a new [commit](/docs/concepts) 
denoted by the timestamp. Look for changes in `_hoodie_commit_time`, `rider`, `driver` fields for the same `_hoodie_record_key`s in previous commit. 
{: .notice--info}

## Incremental query

Hudi also provides capability to obtain a stream of records that changed since given commit timestamp. 
This can be achieved using Hudi's incremental querying and providing a begin time from which changes need to be streamed. 
We do not need to specify endTime, if we want all changes after the given commit (as is the common case). 

```scala
// spark-shell
// reload data
spark.
  read.
  format("hudi").
  load(basePath + "/*/*/*/*").
  createOrReplaceTempView("hudi_trips_snapshot")

val commits = spark.sql("select distinct(_hoodie_commit_time) as commitTime from  hudi_trips_snapshot order by commitTime").map(k => k.getString(0)).take(50)
val beginTime = commits(commits.length - 2) // commit time we are interested in

// incrementally query data
val tripsIncrementalDF = spark.read.format("hudi").
  option(QUERY_TYPE_OPT_KEY, QUERY_TYPE_INCREMENTAL_OPT_VAL).
  option(BEGIN_INSTANTTIME_OPT_KEY, beginTime).
  load(basePath)
tripsIncrementalDF.createOrReplaceTempView("hudi_trips_incremental")

spark.sql("select `_hoodie_commit_time`, fare, begin_lon, begin_lat, ts from  hudi_trips_incremental where fare > 20.0").show()
``` 

This will give all changes that happened after the beginTime commit with the filter of fare > 20.0. The unique thing about this
feature is that it now lets you author streaming pipelines on batch data.
{: .notice--info}

## Point in time query

Lets look at how to query data as of a specific time. The specific time can be represented by pointing endTime to a 
specific commit time and beginTime to "000" (denoting earliest possible commit time). 

```scala
// spark-shell
val beginTime = "000" // Represents all commits > this time.
val endTime = commits(commits.length - 2) // commit time we are interested in

//incrementally query data
val tripsPointInTimeDF = spark.read.format("hudi").
  option(QUERY_TYPE_OPT_KEY, QUERY_TYPE_INCREMENTAL_OPT_VAL).
  option(BEGIN_INSTANTTIME_OPT_KEY, beginTime).
  option(END_INSTANTTIME_OPT_KEY, endTime).
  load(basePath)
tripsPointInTimeDF.createOrReplaceTempView("hudi_trips_point_in_time")
spark.sql("select `_hoodie_commit_time`, fare, begin_lon, begin_lat, ts from hudi_trips_point_in_time where fare > 20.0").show()
```

## Delete data {#deletes}
Delete records for the HoodieKeys passed in.

```scala
// spark-shell
// fetch total records count
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
// fetch two records to be deleted
val ds = spark.sql("select uuid, partitionpath from hudi_trips_snapshot").limit(2)

// issue deletes
val deletes = dataGen.generateDeletes(ds.collectAsList())
val df = spark.read.json(spark.sparkContext.parallelize(deletes, 2))

df.write.format("hudi").
  options(getQuickstartWriteConfigs).
  option(OPERATION_OPT_KEY,"delete").
  option(PRECOMBINE_FIELD_OPT_KEY, "ts").
  option(RECORDKEY_FIELD_OPT_KEY, "uuid").
  option(PARTITIONPATH_FIELD_OPT_KEY, "partitionpath").
  option(TABLE_NAME, tableName).
  mode(Append).
  save(basePath)

// run the same read query as above.
val roAfterDeleteViewDF = spark.
  read.
  format("hudi").
  load(basePath + "/*/*/*/*")

roAfterDeleteViewDF.registerTempTable("hudi_trips_snapshot")
// fetch should return (total - 2) records
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
```
Note: Only `Append` mode is supported for delete operation.

See the [deletion section](/docs/writing_data#deletes) of the writing data page for more details.

## Insert Overwrite Table

Generate some new trips, overwrite the table logically at the Hudi metadata level. The Hudi cleaner will eventually
clean up the previous table snapshot's file groups. This can be faster than deleting the older table and recreating 
in `Overwrite` mode.

```scala
// spark-shell
spark.
  read.format("hudi").
  load(basePath + "/*/*/*/*").
  select("uuid","partitionpath").
  show(10, false)

val inserts = convertToStringList(dataGen.generateInserts(10))
val df = spark.read.json(spark.sparkContext.parallelize(inserts, 2))
df.write.format("hudi").
  options(getQuickstartWriteConfigs).
  option(OPERATION_OPT_KEY,"insert_overwrite_table").
  option(PRECOMBINE_FIELD_OPT_KEY, "ts").
  option(RECORDKEY_FIELD_OPT_KEY, "uuid").
  option(PARTITIONPATH_FIELD_OPT_KEY, "partitionpath").
  option(TABLE_NAME, tableName).
  mode(Append).
  save(basePath)

// Should have different keys now, from query before.
spark.
  read.format("hudi").
  load(basePath + "/*/*/*/*").
  select("uuid","partitionpath").
  show(10, false)

``` 

## Insert Overwrite 

Generate some new trips, overwrite the all the partitions that are present in the input. This operation can be faster
than `upsert` for batch ETL jobs, that are recomputing entire target partitions at once (as opposed to incrementally
updating the target tables). This is because, we are able to bypass indexing, precombining and other repartitioning 
steps in the upsert write path completely.

```scala
// spark-shell
spark.
  read.format("hudi").
  load(basePath + "/*/*/*/*").
  select("uuid","partitionpath").
  sort("partitionpath","uuid").
  show(100, false)

val inserts = convertToStringList(dataGen.generateInserts(10))
val df = spark.
  read.json(spark.sparkContext.parallelize(inserts, 2)).
  filter("partitionpath = 'americas/united_states/san_francisco'")
df.write.format("hudi").
  options(getQuickstartWriteConfigs).
  option(OPERATION_OPT_KEY,"insert_overwrite").
  option(PRECOMBINE_FIELD_OPT_KEY, "ts").
  option(RECORDKEY_FIELD_OPT_KEY, "uuid").
  option(PARTITIONPATH_FIELD_OPT_KEY, "partitionpath").
  option(TABLE_NAME, tableName).
  mode(Append).
  save(basePath)

// Should have different keys now for San Francisco alone, from query before.
spark.
  read.format("hudi").
  load(basePath + "/*/*/*/*").
  select("uuid","partitionpath").
  sort("partitionpath","uuid").
  show(100, false)
```

# Pyspark example
## Setup

Examples below illustrate the same flow above, instead using PySpark.

```python
# pyspark
export PYSPARK_PYTHON=$(which python3)
pyspark \
  --packages org.apache.hudi:hudi-spark-bundle_2.12:0.7.0,org.apache.spark:spark-avro_2.12:3.0.1 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
```

<div className="notice--info">
  <h4>Please note the following: </h4>
<ul>
  <li>spark-avro module needs to be specified in --packages as it is not included with spark-shell by default</li>
  <li>spark-avro and spark versions must match (we have used 3.0.1 for both above)</li>
  <li>we have used hudi-spark-bundle built for scala 2.12 since the spark-avro module used also depends on 2.12. 
         If spark-avro_2.11 is used, correspondingly hudi-spark-bundle_2.11 needs to be used. </li>
</ul>
</div>

Setup table name, base path and a data generator to generate records for this guide.

```python
# pyspark
tableName = "hudi_trips_cow"
basePath = "file:///tmp/hudi_trips_cow"
dataGen = sc._jvm.org.apache.hudi.QuickstartUtils.DataGenerator()
```

The [DataGenerator](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L50) 
can generate sample inserts and updates based on the the sample trip schema [here](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L57)
{: .notice--info}


## Insert data

Generate some new trips, load them into a DataFrame and write the DataFrame into the Hudi table as below.

```python
# pyspark
inserts = sc._jvm.org.apache.hudi.QuickstartUtils.convertToStringList(dataGen.generateInserts(10))
df = spark.read.json(spark.sparkContext.parallelize(inserts, 2))

hudi_options = {
  'hoodie.table.name': tableName,
  'hoodie.datasource.write.recordkey.field': 'uuid',
  'hoodie.datasource.write.partitionpath.field': 'partitionpath',
  'hoodie.datasource.write.table.name': tableName,
  'hoodie.datasource.write.operation': 'upsert',
  'hoodie.datasource.write.precombine.field': 'ts',
  'hoodie.upsert.shuffle.parallelism': 2, 
  'hoodie.insert.shuffle.parallelism': 2
}

df.write.format("hudi"). \
  options(**hudi_options). \
  mode("overwrite"). \
  save(basePath)
```

`mode(Overwrite)` overwrites and recreates the table if it already exists.
You can check the data generated under `/tmp/hudi_trips_cow/<region>/<country>/<city>/`. We provided a record key 
(`uuid` in [schema](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L58)), partition field (`region/county/city`) and combine logic (`ts` in 
[schema](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L58)) to ensure trip records are unique within each partition. For more info, refer to 
[Modeling data stored in Hudi](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=113709185#FAQ-HowdoImodelthedatastoredinHudi)
and for info on ways to ingest data into Hudi, refer to [Writing Hudi Tables](/docs/writing_data).
Here we are using the default write operation : `upsert`. If you have a workload without updates, you can also issue 
`insert` or `bulk_insert` operations which could be faster. To know more, refer to [Write operations](/docs/writing_data#write-operations)
{: .notice--info}

## Query data 

Load the data files into a DataFrame.

```python
# pyspark
tripsSnapshotDF = spark. \
  read. \
  format("hudi"). \
  load(basePath + "/*/*/*/*")
# load(basePath) use "/partitionKey=partitionValue" folder structure for Spark auto partition discovery

tripsSnapshotDF.createOrReplaceTempView("hudi_trips_snapshot")

spark.sql("select fare, begin_lon, begin_lat, ts from  hudi_trips_snapshot where fare > 20.0").show()
spark.sql("select _hoodie_commit_time, _hoodie_record_key, _hoodie_partition_path, rider, driver, fare from  hudi_trips_snapshot").show()
```

This query provides snapshot querying of the ingested data. Since our partition path (`region/country/city`) is 3 levels nested 
from base path we ve used `load(basePath + "/*/*/*/*")`. 
Refer to [Table types and queries](/docs/concepts#table-types--queries) for more info on all table types and query types supported.
{: .notice--info}

## Update data

This is similar to inserting new data. Generate updates to existing trips using the data generator, load into a DataFrame 
and write DataFrame into the hudi table.

```python
# pyspark
updates = sc._jvm.org.apache.hudi.QuickstartUtils.convertToStringList(dataGen.generateUpdates(10))
df = spark.read.json(spark.sparkContext.parallelize(updates, 2))
df.write.format("hudi"). \
  options(**hudi_options). \
  mode("append"). \
  save(basePath)
```

Notice that the save mode is now `Append`. In general, always use append mode unless you are trying to create the table for the first time.
[Querying](#query-data) the data again will now show updated trips. Each write operation generates a new [commit](/docs/concepts) 
denoted by the timestamp. Look for changes in `_hoodie_commit_time`, `rider`, `driver` fields for the same `_hoodie_record_key`s in previous commit. 
{: .notice--info}

## Incremental query

Hudi also provides capability to obtain a stream of records that changed since given commit timestamp. 
This can be achieved using Hudi's incremental querying and providing a begin time from which changes need to be streamed. 
We do not need to specify endTime, if we want all changes after the given commit (as is the common case). 

```python
# pyspark
# reload data
spark. \
  read. \
  format("hudi"). \
  load(basePath + "/*/*/*/*"). \
  createOrReplaceTempView("hudi_trips_snapshot")

commits = list(map(lambda row: row[0], spark.sql("select distinct(_hoodie_commit_time) as commitTime from  hudi_trips_snapshot order by commitTime").limit(50).collect()))
beginTime = commits[len(commits) - 2] # commit time we are interested in

# incrementally query data
incremental_read_options = {
  'hoodie.datasource.query.type': 'incremental',
  'hoodie.datasource.read.begin.instanttime': beginTime,
}

tripsIncrementalDF = spark.read.format("hudi"). \
  options(**incremental_read_options). \
  load(basePath)
tripsIncrementalDF.createOrReplaceTempView("hudi_trips_incremental")

spark.sql("select `_hoodie_commit_time`, fare, begin_lon, begin_lat, ts from  hudi_trips_incremental where fare > 20.0").show()
```

This will give all changes that happened after the beginTime commit with the filter of fare > 20.0. The unique thing about this
feature is that it now lets you author streaming pipelines on batch data.
{: .notice--info}

## Point in time query

Lets look at how to query data as of a specific time. The specific time can be represented by pointing endTime to a 
specific commit time and beginTime to "000" (denoting earliest possible commit time). 

```python
# pyspark
beginTime = "000" # Represents all commits > this time.
endTime = commits[len(commits) - 2]

# query point in time data
point_in_time_read_options = {
  'hoodie.datasource.query.type': 'incremental',
  'hoodie.datasource.read.end.instanttime': endTime,
  'hoodie.datasource.read.begin.instanttime': beginTime
}

tripsPointInTimeDF = spark.read.format("hudi"). \
  options(**point_in_time_read_options). \
  load(basePath)

tripsPointInTimeDF.createOrReplaceTempView("hudi_trips_point_in_time")
spark.sql("select `_hoodie_commit_time`, fare, begin_lon, begin_lat, ts from hudi_trips_point_in_time where fare > 20.0").show()
```

## Delete data {#deletes}
Delete records for the HoodieKeys passed in.

Note: Only `Append` mode is supported for delete operation.

```python
# pyspark
# fetch total records count
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
# fetch two records to be deleted
ds = spark.sql("select uuid, partitionpath from hudi_trips_snapshot").limit(2)

# issue deletes
hudi_delete_options = {
  'hoodie.table.name': tableName,
  'hoodie.datasource.write.recordkey.field': 'uuid',
  'hoodie.datasource.write.partitionpath.field': 'partitionpath',
  'hoodie.datasource.write.table.name': tableName,
  'hoodie.datasource.write.operation': 'delete',
  'hoodie.datasource.write.precombine.field': 'ts',
  'hoodie.upsert.shuffle.parallelism': 2, 
  'hoodie.insert.shuffle.parallelism': 2
}

from pyspark.sql.functions import lit
deletes = list(map(lambda row: (row[0], row[1]), ds.collect()))
df = spark.sparkContext.parallelize(deletes).toDF(['uuid', 'partitionpath']).withColumn('ts', lit(0.0))
df.write.format("hudi"). \
  options(**hudi_delete_options). \
  mode("append"). \
  save(basePath)

# run the same read query as above.
roAfterDeleteViewDF = spark. \
  read. \
  format("hudi"). \
  load(basePath + "/*/*/*/*") 
roAfterDeleteViewDF.registerTempTable("hudi_trips_snapshot")
# fetch should return (total - 2) records
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
```

See the [deletion section](/docs/writing_data#deletes) of the writing data page for more details.


## Where to go from here?

You can also do the quickstart by [building hudi yourself](https://github.com/apache/hudi#building-apache-hudi-from-source), 
and using `--jars <path to hudi_code>/packaging/hudi-spark-bundle/target/hudi-spark-bundle_2.1?-*.*.*-SNAPSHOT.jar` in the spark-shell command above
instead of `--packages org.apache.hudi:hudi-spark-bundle_2.12:0.7.0`. Hudi also supports scala 2.12. Refer [build with scala 2.12](https://github.com/apache/hudi#build-with-scala-212)
for more info.

Also, we used Spark here to show case the capabilities of Hudi. However, Hudi can support multiple table types/query types and 
Hudi tables can be queried from query engines like Hive, Spark, Presto and much more. We have put together a 
[demo video](https://www.youtube.com/watch?v=VhNgUsxdrD0) that show cases all of this on a docker based setup with all 
dependent systems running locally. We recommend you replicate the same setup and run the demo yourself, by following 
steps [here](/docs/docker_demo) to get a taste for it. Also, if you are looking for ways to migrate your existing data 
to Hudi, refer to [migration guide](/docs/migration_guide). 
