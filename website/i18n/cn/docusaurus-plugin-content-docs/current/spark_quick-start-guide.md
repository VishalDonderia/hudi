---
title: "Spark Guide"
toc: true
last_modified_at: 2019-12-30T15:59:57-04:00
language: cn
---

本指南通过使用spark-shell简要介绍了Hudi功能。使用Spark数据源，我们将通过代码段展示如何插入和更新Hudi的默认存储类型数据集：
[写时复制](/cn/docs/concepts#copy-on-write-storage)。每次写操作之后，我们还将展示如何读取快照和增量数据。 

## 设置spark-shell
Hudi适用于Spark-2.4.3+ & Spark 3.x 版本。您可以按照[此处](https://spark.apache.org/downloads)的说明设置spark。
在提取的目录中，使用spark-shell运行Hudi：

```scala
// spark-shell for spark 3
spark-shell \
  --packages org.apache.hudi:hudi-spark3-bundle_2.12:0.8.0,org.apache.spark:spark-avro_2.12:3.0.1 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
// spark-shell for spark 2 with scala 2.12
spark-shell \
  --packages org.apache.hudi:hudi-spark-bundle_2.12:0.8.0,org.apache.spark:spark-avro_2.12:2.4.4 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
// spark-shell for spark 2 with scala 2.11
spark-shell \
  --packages org.apache.hudi:hudi-spark-bundle_2.11:0.8.0,org.apache.spark:spark-avro_2.11:2.4.4 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
```


设置表名、基本路径和数据生成器来为本指南生成记录。

```scala
import org.apache.hudi.QuickstartUtils._
import scala.collection.JavaConversions._
import org.apache.spark.sql.SaveMode._
import org.apache.hudi.DataSourceReadOptions._
import org.apache.hudi.DataSourceWriteOptions._
import org.apache.hudi.config.HoodieWriteConfig._

val tableName = "hudi_cow_table"
val basePath = "file:///tmp/hudi_cow_table"
val dataGen = new DataGenerator
```

[数据生成器](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L50)
可以基于[行程样本模式](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L57)
生成插入和更新的样本。

## 插入数据 {#inserts}
生成一些新的行程样本，将其加载到DataFrame中，然后将DataFrame写入Hudi数据集中，如下所示。

```scala
val inserts = convertToStringList(dataGen.generateInserts(10))
val df = spark.read.json(spark.sparkContext.parallelize(inserts, 2))
df.write.format("org.apache.hudi").
    options(getQuickstartWriteConfigs).
    option(PRECOMBINE_FIELD_OPT_KEY, "ts").
    option(RECORDKEY_FIELD_OPT_KEY, "uuid").
    option(PARTITIONPATH_FIELD_OPT_KEY, "partitionpath").
    option(TABLE_NAME, tableName).
    mode(Overwrite).
    save(basePath);
```

`mode(Overwrite)`覆盖并重新创建数据集(如果已经存在)。
您可以检查在`/tmp/hudi_cow_table/<region>/<country>/<city>/`下生成的数据。我们提供了一个记录键
([schema](#sample-schema)中的`uuid`)，分区字段(`region/country/city`）和组合逻辑([schema](#sample-schema)中的`ts`)
以确保行程记录在每个分区中都是唯一的。更多信息请参阅
[对Hudi中的数据进行建模](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=113709185#FAQ-HowdoImodelthedatastoredinHudi)，
有关将数据提取到Hudi中的方法的信息，请参阅[写入Hudi数据集](/cn/docs/writing_data)。
这里我们使用默认的写操作：`插入更新`。 如果您的工作负载没有`更新`，也可以使用更快的`插入`或`批量插入`操作。
想了解更多信息，请参阅[写操作](/cn/docs/writing_data#write-operations)

## 查询数据 {#query}

将数据文件加载到DataFrame中。

```scala
val roViewDF = spark.
    read.
    format("org.apache.hudi").
    load(basePath + "/*/*/*/*")
    //load(basePath) 如果使用 "/partitionKey=partitionValue" 文件夹命名格式，Spark将自动识别分区信息

roViewDF.registerTempTable("hudi_ro_table")
spark.sql("select fare, begin_lon, begin_lat, ts from  hudi_ro_table where fare > 20.0").show()
spark.sql("select _hoodie_commit_time, _hoodie_record_key, _hoodie_partition_path, rider, driver, fare from  hudi_ro_table").show()
```

该查询提供已提取数据的读取优化视图。由于我们的分区路径(`region/country/city`)是嵌套的3个级别
从基本路径开始，我们使用了`load(basePath + "/*/*/*/*")`。
有关支持的所有存储类型和视图的更多信息，请参考[存储类型和视图](/cn/docs/concepts#storage-types--views)。

## 更新数据 {#updates}

这类似于插入新数据。使用数据生成器生成对现有行程的更新，加载到DataFrame中并将DataFrame写入hudi数据集。

```scala
val updates = convertToStringList(dataGen.generateUpdates(10))
val df = spark.read.json(spark.sparkContext.parallelize(updates, 2));
df.write.format("org.apache.hudi").
    options(getQuickstartWriteConfigs).
    option(PRECOMBINE_FIELD_OPT_KEY, "ts").
    option(RECORDKEY_FIELD_OPT_KEY, "uuid").
    option(PARTITIONPATH_FIELD_OPT_KEY, "partitionpath").
    option(TABLE_NAME, tableName).
    mode(Append).
    save(basePath);
```

注意，保存模式现在为`追加`。通常，除非您是第一次尝试创建数据集，否则请始终使用追加模式。
[查询](#query)现在再次查询数据将显示更新的行程。每个写操作都会生成一个新的由时间戳表示的[commit](/cn/docs/concepts)
。在之前提交的相同的`_hoodie_record_key`中寻找`_hoodie_commit_time`, `rider`, `driver`字段变更。

## 增量查询

Hudi还提供了获取给定提交时间戳以来已更改的记录流的功能。
这可以通过使用Hudi的增量视图并提供所需更改的开始时间来实现。
如果我们需要给定提交之后的所有更改(这是常见的情况)，则无需指定结束时间。

```scala
// reload data
spark.
    read.
    format("org.apache.hudi").
    load(basePath + "/*/*/*/*").
    createOrReplaceTempView("hudi_ro_table")

val commits = spark.sql("select distinct(_hoodie_commit_time) as commitTime from  hudi_ro_table order by commitTime").map(k => k.getString(0)).take(50)
val beginTime = commits(commits.length - 2) // commit time we are interested in

// 增量查询数据
val incViewDF = spark.
    read.
    format("org.apache.hudi").
    option(VIEW_TYPE_OPT_KEY, VIEW_TYPE_INCREMENTAL_OPT_VAL).
    option(BEGIN_INSTANTTIME_OPT_KEY, beginTime).
    load(basePath);
incViewDF.registerTempTable("hudi_incr_table")
spark.sql("select `_hoodie_commit_time`, fare, begin_lon, begin_lat, ts from  hudi_incr_table where fare > 20.0").show()
```

这将提供在开始时间提交之后发生的所有更改，其中包含票价大于20.0的过滤器。关于此功能的独特之处在于，它现在使您可以在批量数据上创作流式管道。

## 特定时间点查询

让我们看一下如何查询特定时间的数据。可以通过将结束时间指向特定的提交时间，将开始时间指向"000"(表示最早的提交时间)来表示特定时间。

```scala
val beginTime = "000" // Represents all commits > this time.
val endTime = commits(commits.length - 2) // commit time we are interested in

// 增量查询数据
val incViewDF = spark.read.format("org.apache.hudi").
    option(VIEW_TYPE_OPT_KEY, VIEW_TYPE_INCREMENTAL_OPT_VAL).
    option(BEGIN_INSTANTTIME_OPT_KEY, beginTime).
    option(END_INSTANTTIME_OPT_KEY, endTime).
    load(basePath);
incViewDF.registerTempTable("hudi_incr_table")
spark.sql("select `_hoodie_commit_time`, fare, begin_lon, begin_lat, ts from  hudi_incr_table where fare > 20.0").show()
```

## 删除数据 {#deletes}
删除传入的 HoodieKeys 的记录。

```scala
// spark-shell
// 获取记录总数
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
// 拿到两条将要删除的数据
val ds = spark.sql("select uuid, partitionpath from hudi_trips_snapshot").limit(2)

// 执行删除
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

// 向之前一样运行查询
val roAfterDeleteViewDF = spark.
  read.
  format("hudi").
  load(basePath + "/*/*/*/*")

roAfterDeleteViewDF.registerTempTable("hudi_trips_snapshot")
// 应返回 (total - 2) 条记录
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
```
注意: 删除操作只支持 `Append` 模式。



请查阅写数据页的[删除部分](/cn/docs/writing_data#deletes) 查看更多信息.


# Pyspark 示例
## 设置spark-shell

Hudi适用于Spark-2.4.3+ & Spark 3.x 版本。您可以按照[此处](https://spark.apache.org/downloads)的说明设置spark。
在提取的目录中，使用spark-shell运行Hudi：

```python
# pyspark
export PYSPARK_PYTHON=$(which python3)
# for spark3
pyspark \
  --packages org.apache.hudi:hudi-spark3-bundle_2.12:0.8.0,org.apache.spark:spark-avro_2.12:3.0.1 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
# for spark2 with scala 2.12
pyspark \
  --packages org.apache.hudi:hudi-spark-bundle_2.12:0.8.0,org.apache.spark:spark-avro_2.12:2.4.4 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
# for spark2 with scala 2.11
pyspark \
  --packages org.apache.hudi:hudi-spark-bundle_2.11:0.8.0,org.apache.spark:spark-avro_2.11:2.4.4 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
```

<div className="notice--info">
  <h4>请注意以下事项: </h4>
<ul>
  <li>需要通过 --packages 指定 spark-avro, 因为默认情况下 spark-shell 不包含该模块</li>
  <li>spark-avro 和 spark 的版本必须匹配 (上面两个我们都使用了2.4.4)</li>
  <li>我们使用了基于 scala 2.11 构建的 hudi-spark-bundle, 因为使用的 spark-avro 也是基于 scala 2.11的. 
         如果使用了 spark-avro_2.12, 相应的, 需要使用 hudi-spark-bundle_2.12. </li>
</ul>
</div>

设置表名、基本路径和数据生成器来为本指南生成记录。

```python
# pyspark
tableName = "hudi_trips_cow"
basePath = "file:///tmp/hudi_trips_cow"
dataGen = sc._jvm.org.apache.hudi.QuickstartUtils.DataGenerator()
```

[数据生成器](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L50)
可以基于[行程样本模式](https://github.com/apache/hudi/blob/master/hudi-spark/src/main/java/org/apache/hudi/QuickstartUtils.java#L57)
生成插入和更新的样本。
{: .notice--info}


## 插入数据 {#inserts}

生成一些新的行程样本，将其加载到DataFrame中，然后将DataFrame写入Hudi数据集中，如下所示。

```python
# pyspark
inserts = sc._jvm.org.apache.hudi.QuickstartUtils.convertToStringList(dataGen.generateInserts(10))
df = spark.read.json(spark.sparkContext.parallelize(inserts, 2))

hudi_options = {
  'hoodie.table.name': tableName,
  'hoodie.datasource.write.recordkey.field': 'uuid',
  'hoodie.datasource.write.partitionpath.field': 'partitionpath',
  'hoodie.datasource.write.table.name': tableName,
  'hoodie.datasource.write.operation': 'insert',
  'hoodie.datasource.write.precombine.field': 'ts',
  'hoodie.upsert.shuffle.parallelism': 2, 
  'hoodie.insert.shuffle.parallelism': 2
}

df.write.format("hudi"). \
  options(**hudi_options). \
  mode("overwrite"). \
  save(basePath)
```

`mode(Overwrite)` 覆盖并重新创建数据集(如果已经存在)。
您可以检查在`/tmp/hudi_cow_table/<region>/<country>/<city>/`下生成的数据。我们提供了一个记录键
([schema](#sample-schema)中的`uuid`)，分区字段(`region/country/city`）和组合逻辑([schema](#sample-schema)中的`ts`)
以确保行程记录在每个分区中都是唯一的。更多信息请参阅
[对Hudi中的数据进行建模](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=113709185#FAQ-HowdoImodelthedatastoredinHudi)，
有关将数据提取到Hudi中的方法的信息，请参阅[写入Hudi数据集](/cn/docs/writing_data)。
这里我们使用默认的写操作：`插入更新`。 如果您的工作负载没有`更新`，也可以使用更快的`插入`或`批量插入`操作。
想了解更多信息，请参阅[写操作](/cn/docs/writing_data#write-operations)
{: .notice--info}

## 查询数据 {#query}

将数据文件加载到DataFrame中。

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

该查询提供已提取数据的读取优化视图。由于我们的分区路径(`region/country/city`)是嵌套的3个级别
从基本路径开始，我们使用了`load(basePath + "/*/*/*/*")`。
有关支持的所有存储类型和视图的更多信息，请参考[存储类型和视图](/cn/docs/concepts#storage-types--views)。
{: .notice--info}

## 更新数据 {#updates}

这类似于插入新数据。使用数据生成器生成对现有行程的更新，加载到DataFrame中并将DataFrame写入hudi数据集。

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

注意，保存模式现在为`追加`。通常，除非您是第一次尝试创建数据集，否则请始终使用追加模式。
[查询](#query)现在再次查询数据将显示更新的行程。每个写操作都会生成一个新的由时间戳表示的[commit](/cn/docs/concepts)
。在之前提交的相同的`_hoodie_record_key`中寻找`_hoodie_commit_time`, `rider`, `driver`字段变更。
{: .notice--info}

## 增量查询

Hudi还提供了获取给定提交时间戳以来已更改的记录流的功能。
这可以通过使用Hudi的增量视图并提供所需更改的开始时间来实现。
如果我们需要给定提交之后的所有更改(这是常见的情况)，则无需指定结束时间。

```python
# pyspark
# 加载数据
spark. \
  read. \
  format("hudi"). \
  load(basePath + "/*/*/*/*"). \
  createOrReplaceTempView("hudi_trips_snapshot")

commits = list(map(lambda row: row[0], spark.sql("select distinct(_hoodie_commit_time) as commitTime from  hudi_trips_snapshot order by commitTime").limit(50).collect()))
beginTime = commits[len(commits) - 2] # commit time we are interested in

# 增量的查询数据
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

这将提供在开始时间提交之后发生的所有更改，其中包含票价大于20.0的过滤器。关于此功能的独特之处在于，它现在使您可以在批量数据上创作流式管道。
{: .notice--info}

## 特定时间点查询

让我们看一下如何查询特定时间的数据。可以通过将结束时间指向特定的提交时间，将开始时间指向"000"(表示最早的提交时间)来表示特定时间。

```python
# pyspark
beginTime = "000" # 代表所有大于该时间的 commits.
endTime = commits[len(commits) - 2]

# 特定时间查询
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

## 删除数据 {#deletes}
删除传入的 HoodieKeys 的记录。

注意: 删除操作只支持 `Append` 模式。

```python
# pyspark
# 获取记录总数
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
# 拿到两条将被删除的记录
ds = spark.sql("select uuid, partitionpath from hudi_trips_snapshot").limit(2)

# 执行删除
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

# 向之前一样运行查询
roAfterDeleteViewDF = spark. \
  read. \
  format("hudi"). \
  load(basePath + "/*/*/*/*") 
roAfterDeleteViewDF.registerTempTable("hudi_trips_snapshot")
# 应返回 (total - 2) 条记录
spark.sql("select uuid, partitionpath from hudi_trips_snapshot").count()
```

请查阅写数据页的[删除部分](/cn/docs/writing_data#deletes) 查看更多信息.


## 从这开始下一步？

您也可以通过[自己构建hudi](https://github.com/apache/hudi#building-apache-hudi-from-source)来快速开始，
并在spark-shell命令中使用`--jars <path to hudi_code>/packaging/hudi-spark-bundle/target/hudi-spark-bundle_2.1?-*.*.*-SNAPSHOT.jar`，
而不是`--packages org.apache.hudi:hudi-spark3-bundle_2.12:0.8.0`


这里我们使用Spark演示了Hudi的功能。但是，Hudi可以支持多种存储类型/视图，并且可以从Hive，Spark，Presto等查询引擎中查询Hudi数据集。
我们制作了一个基于Docker设置、所有依赖系统都在本地运行的[演示视频](https://www.youtube.com/watch?v=VhNgUsxdrD0)，
我们建议您复制相同的设置然后按照[这里](/cn/docs/docker_demo)的步骤自己运行这个演示。
另外，如果您正在寻找将现有数据迁移到Hudi的方法，请参考[迁移指南](/cn/docs/migration_guide)。
