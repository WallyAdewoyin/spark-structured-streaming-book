# KafkaSourceProvider &mdash; Data Source Provider for Apache Kafka

[[shortName]]
`KafkaSourceProvider` is a `DataSourceRegister` and registers a developer-friendly alias for *kafka* data source format in Spark Structured Streaming.

TIP: Read up on https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-DataSourceRegister.html[DataSourceRegister] in https://bit.ly/spark-sql-internals[The Internals of Spark SQL] book.

`KafkaSourceProvider` supports <<micro-batch-stream-processing.md#, micro-batch stream processing>> (through <<spark-sql-streaming-MicroBatchReadSupport.md#, MicroBatchReadSupport>> contract) and <<createMicroBatchReader, creates a specialized KafkaMicroBatchReader>>.

`KafkaSourceProvider` requires the following options (that you can set using `option` method of <<spark-sql-streaming-DataStreamReader.md#, DataStreamReader>> or [DataStreamWriter](DataStreamWriter.md)):

* Exactly one of the following options: <<spark-sql-streaming-kafka-data-source.md#subscribe, subscribe>>, <<spark-sql-streaming-kafka-data-source.md#subscribePattern, subscribePattern>> or <<spark-sql-streaming-kafka-data-source.md#assign, assign>>

* <<spark-sql-streaming-kafka-data-source.md#kafka.bootstrap.servers, kafka.bootstrap.servers>>

TIP: Refer to <<spark-sql-streaming-kafka-data-source.md#options, Kafka Data Source's Options>> for the supported configuration options.

Internally, `KafkaSourceProvider` sets the <<kafkaParamsForExecutors-properties, properties for Kafka Consumers on executors>> (that are passed on to `InternalKafkaConsumer` when requested to create a Kafka consumer with a single `TopicPartition` manually assigned).

[[kafkaParamsForExecutors-properties]]
.KafkaSourceProvider's Properties for Kafka Consumers on Executors
[cols="1m,1m,2",options="header",width="100%"]
|===
| ConsumerConfig's Key
| Value
| Description

| KEY_DESERIALIZER_CLASS_CONFIG
| ByteArrayDeserializer
a| [[KEY_DESERIALIZER_CLASS_CONFIG]] FIXME

| VALUE_DESERIALIZER_CLASS_CONFIG
| ByteArrayDeserializer
a| [[VALUE_DESERIALIZER_CLASS_CONFIG]] FIXME

| AUTO_OFFSET_RESET_CONFIG
| none
a| [[AUTO_OFFSET_RESET_CONFIG]] FIXME

| GROUP_ID_CONFIG
| <<uniqueGroupId, uniqueGroupId>>-executor
a| [[GROUP_ID_CONFIG]] FIXME

| ENABLE_AUTO_COMMIT_CONFIG
| false
a| [[ENABLE_AUTO_COMMIT_CONFIG]] FIXME

| RECEIVE_BUFFER_CONFIG
| 65536
a| [[RECEIVE_BUFFER_CONFIG]] Only when not set in the <<specifiedKafkaParams, specifiedKafkaParams>> already

|===

[[logging]]
[TIP]
====
Enable `ALL` logging levels for `org.apache.spark.sql.kafka010.KafkaSourceProvider` logger to see what happens inside.

Add the following line to `conf/log4j.properties`:

```
log4j.logger.org.apache.spark.sql.kafka010.KafkaSourceProvider=ALL
```

Refer to <<spark-sql-streaming-logging.md#, Logging>>.
====

=== [[createSource]] Creating Streaming Source -- `createSource` Method

[source, scala]
----
createSource(
  sqlContext: SQLContext,
  metadataPath: String,
  schema: Option[StructType],
  providerName: String,
  parameters: Map[String, String]): Source
----

`createSource` is part of the [StreamSourceProvider](StreamSourceProvider.md#createSource) abstraction.

`createSource` first <<validateStreamOptions, validates stream options>>.

`createSource`...FIXME

=== [[validateGeneralOptions]] Validating General Options For Batch And Streaming Queries -- `validateGeneralOptions` Internal Method

[source, scala]
----
validateGeneralOptions(parameters: Map[String, String]): Unit
----

NOTE: Parameters are case-insensitive, i.e. `OptioN` and `option` are equal.

`validateGeneralOptions` makes sure that exactly one topic subscription strategy is used in `parameters` and can be:

* `subscribe`
* `subscribepattern`
* `assign`

`validateGeneralOptions` reports an `IllegalArgumentException` when there is no subscription strategy in use or there are more than one strategies used.

`validateGeneralOptions` makes sure that the value of subscription strategies meet the requirements:

* `assign` strategy starts with `{` (the opening curly brace)
* `subscribe` strategy has at least one topic (in a comma-separated list of topics)
* `subscribepattern` strategy has the pattern defined

`validateGeneralOptions` makes sure that `group.id` has not been specified and reports an `IllegalArgumentException` otherwise.

```
Kafka option 'group.id' is not supported as user-specified consumer groups are not used to track offsets.
```

`validateGeneralOptions` makes sure that `auto.offset.reset` has not been specified and reports an `IllegalArgumentException` otherwise.

[options="wrap"]
----
Kafka option 'auto.offset.reset' is not supported.
Instead set the source option 'startingoffsets' to 'earliest' or 'latest' to specify where to start. Structured Streaming manages which offsets are consumed internally, rather than relying on the kafkaConsumer to do it. This will ensure that no data is missed when new topics/partitions are dynamically subscribed. Note that 'startingoffsets' only applies when a new Streaming query is started, and
that resuming will always pick up from where the query left off. See the docs for more details.
----

`validateGeneralOptions` makes sure that the following options have not been specified and reports an `IllegalArgumentException` otherwise:

* `kafka.key.deserializer`
* `kafka.value.deserializer`
* `kafka.enable.auto.commit`
* `kafka.interceptor.classes`

In the end, `validateGeneralOptions` makes sure that `kafka.bootstrap.servers` option was specified and reports an `IllegalArgumentException` otherwise.

```
Option 'kafka.bootstrap.servers' must be specified for configuring Kafka consumer
```

NOTE: `validateGeneralOptions` is used when `KafkaSourceProvider` validates options for <<validateStreamOptions, streaming>> and <<validateBatchOptions, batch>> queries.

=== [[strategy]] Creating ConsumerStrategy -- `strategy` Internal Method

[source, scala]
----
strategy(caseInsensitiveParams: Map[String, String])
----

Internally, `strategy` finds the keys in the input `caseInsensitiveParams` that are one of the following and creates a corresponding spark-sql-streaming-ConsumerStrategy.md[ConsumerStrategy].

.KafkaSourceProvider.strategy's Key to ConsumerStrategy Conversion
[cols="1m,2",options="header",width="100%"]
|===
| Key
| ConsumerStrategy

| assign
a| spark-sql-streaming-ConsumerStrategy.md#AssignStrategy[AssignStrategy] with Kafka's http://kafka.apache.org/0110/javadoc/org/apache/kafka/common/TopicPartition.html[TopicPartitions].

---

`strategy` uses `JsonUtils.partitions` method to parse a JSON with topic names and partitions, e.g.

```
{"topicA":[0,1],"topicB":[0,1]}
```

The topic names and partitions are mapped directly to Kafka's `TopicPartition` objects.

| subscribe
a| spark-sql-streaming-ConsumerStrategy.md#SubscribeStrategy[SubscribeStrategy] with topic names

---

`strategy` extracts topic names from a comma-separated string, e.g.

```
topic1,topic2,topic3
```

| subscribepattern
a| spark-sql-streaming-ConsumerStrategy.md#SubscribePatternStrategy[SubscribePatternStrategy] with topic subscription regex pattern (that uses Java's http://docs.oracle.com/javase/8/docs/api/java/util/regex/Pattern.html[java.util.regex.Pattern] for the pattern), e.g.

```
topic\d
```

|===

[NOTE]
====
`strategy` is used when:

* `KafkaSourceProvider` <<createSource, creates a KafkaOffsetReader for KafkaSource>>.

* `KafkaSourceProvider` creates a KafkaRelation (using `createRelation` method).
====

=== [[sourceSchema]] Describing Streaming Source with Name and Schema -- `sourceSchema` Method

[source, scala]
----
sourceSchema(
  sqlContext: SQLContext,
  schema: Option[StructType],
  providerName: String,
  parameters: Map[String, String]): (String, StructType)
----

`sourceSchema` gives the <<shortName, short name>> (i.e. `kafka`) and the spark-sql-streaming-KafkaOffsetReader.md#kafkaSchema[fixed schema].

Internally, `sourceSchema` <<validateStreamOptions, validates Kafka options>> and makes sure that the optional input `schema` is indeed undefined.

When the input `schema` is defined, `sourceSchema` reports a `IllegalArgumentException`.

```text
Kafka source has a fixed schema and cannot be set with a custom one
```

`sourceSchema` is part of the [StreamSourceProvider](StreamSourceProvider.md#sourceSchema) abstraction.

=== [[validateStreamOptions]] Validating Kafka Options for Streaming Queries -- `validateStreamOptions` Internal Method

[source, scala]
----
validateStreamOptions(caseInsensitiveParams: Map[String, String]): Unit
----

Firstly, `validateStreamOptions` makes sure that `endingoffsets` option is not used. Otherwise, `validateStreamOptions` reports a `IllegalArgumentException`.

```
ending offset not valid in streaming queries
```

`validateStreamOptions` then <<validateGeneralOptions, validates the general options>>.

NOTE: `validateStreamOptions` is used when `KafkaSourceProvider` is requested the <<sourceSchema, schema for Kafka source>> and to <<createSource, create a KafkaSource>>.

=== [[createContinuousReader]] Creating ContinuousReader for Continuous Stream Processing -- `createContinuousReader` Method

[source, scala]
----
createContinuousReader(
  schema: Optional[StructType],
  metadataPath: String,
  options: DataSourceOptions): KafkaContinuousReader
----

NOTE: `createContinuousReader` is part of the <<spark-sql-streaming-ContinuousReadSupport.md#createContinuousReader, ContinuousReadSupport Contract>> to create a <<spark-sql-streaming-ContinuousReader.md#, ContinuousReader>>.

`createContinuousReader`...FIXME

=== [[getKafkaOffsetRangeLimit]] Converting Configuration Options to KafkaOffsetRangeLimit -- `getKafkaOffsetRangeLimit` Object Method

[source, scala]
----
getKafkaOffsetRangeLimit(
  params: Map[String, String],
  offsetOptionKey: String,
  defaultOffsets: KafkaOffsetRangeLimit): KafkaOffsetRangeLimit
----

`getKafkaOffsetRangeLimit` finds the given `offsetOptionKey` in the `params` and does the following conversion:

* *latest* becomes <<spark-sql-streaming-KafkaOffsetRangeLimit.md#LatestOffsetRangeLimit, LatestOffsetRangeLimit>>

* *earliest* becomes <<spark-sql-streaming-KafkaOffsetRangeLimit.md#EarliestOffsetRangeLimit, EarliestOffsetRangeLimit>>

* A JSON-formatted text becomes <<spark-sql-streaming-KafkaOffsetRangeLimit.md#SpecificOffsetRangeLimit, SpecificOffsetRangeLimit>>

* When the given `offsetOptionKey` is not found, `getKafkaOffsetRangeLimit` returns the given `defaultOffsets`

NOTE: `getKafkaOffsetRangeLimit` is used when `KafkaSourceProvider` is requested to <<createSource, createSource>>, <<createMicroBatchReader, createMicroBatchReader>>, <<createContinuousReader, createContinuousReader>>, <<createRelation, createRelation>>, and <<validateBatchOptions, validateBatchOptions>>.

=== [[createMicroBatchReader]] Creating MicroBatchReader for Micro-Batch Stream Processing -- `createMicroBatchReader` Method

[source, scala]
----
createMicroBatchReader(
  schema: Optional[StructType],
  metadataPath: String,
  options: DataSourceOptions): KafkaMicroBatchReader
----

NOTE: `createMicroBatchReader` is part of the <<spark-sql-streaming-MicroBatchReadSupport.md#createMicroBatchReader, MicroBatchReadSupport Contract>> to create a <<spark-sql-streaming-MicroBatchReader.md#, MicroBatchReader>> in <<micro-batch-stream-processing.md#, Micro-Batch Stream Processing>>.

`createMicroBatchReader` <<validateStreamOptions, validateStreamOptions>> (in the given `DataSourceOptions`).

`createMicroBatchReader` generates a unique group ID of the format *spark-kafka-source-[randomUUID]-[metadataPath_hashCode]* (to make sure that a new streaming query creates a new consumer group).

`createMicroBatchReader` finds all the parameters (in the given `DataSourceOptions`) that start with *kafka.* prefix, removes the prefix, and creates the current Kafka parameters.

`createMicroBatchReader` creates a <<spark-sql-streaming-KafkaOffsetReader.md#, KafkaOffsetReader>> with the following:

* <<strategy, strategy>> (in the given `DataSourceOptions`)

* <<kafkaParamsForDriver, Properties for Kafka consumers on the driver>> (given the current Kafka parameters, i.e. without *kafka.* prefix)

* The given `DataSourceOptions`

* *spark-kafka-source-[randomUUID]-[metadataPath_hashCode]-driver* for the `driverGroupIdPrefix`

In the end, `createMicroBatchReader` creates a <<spark-sql-streaming-KafkaMicroBatchReader.md#, KafkaMicroBatchReader>> with the following:

* the `KafkaOffsetReader`

* <<kafkaParamsForExecutors, Properties for Kafka consumers on executors>> (given the current Kafka parameters, i.e. without *kafka.* prefix) and the unique group ID (`spark-kafka-source-[randomUUID]-[metadataPath_hashCode]-driver`)

* The given `DataSourceOptions` and the `metadataPath`

* <<getKafkaOffsetRangeLimit, Starting stream offsets>> (<<spark-sql-streaming-kafka-data-source.md#startingOffsets, startingOffsets>> option with the default of `LatestOffsetRangeLimit` offsets)

* <<failOnDataLoss, failOnDataLoss configuration property>>

=== [[createRelation]] Creating BaseRelation -- `createRelation` Method

[source, scala]
----
createRelation(
  sqlContext: SQLContext,
  parameters: Map[String, String]): BaseRelation
----

NOTE: `createRelation` is part of the https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-RelationProvider.html[RelationProvider] contract to create a `BaseRelation`.

`createRelation`...FIXME

=== [[validateBatchOptions]] Validating Configuration Options for Batch Processing -- `validateBatchOptions` Internal Method

[source, scala]
----
validateBatchOptions(caseInsensitiveParams: Map[String, String]): Unit
----

`validateBatchOptions`...FIXME

NOTE: `validateBatchOptions` is used exclusively when `KafkaSourceProvider` is requested to <<createSource, createSource>>.

=== [[kafkaParamsForDriver]] `kafkaParamsForDriver` Method

[source, scala]
----
kafkaParamsForDriver(specifiedKafkaParams: Map[String, String]): Map[String, Object]
----

`kafkaParamsForDriver`...FIXME

NOTE: `kafkaParamsForDriver` is used when...FIXME

=== [[kafkaParamsForExecutors]] `kafkaParamsForExecutors` Method

[source, scala]
----
kafkaParamsForExecutors(
  specifiedKafkaParams: Map[String, String],
  uniqueGroupId: String): Map[String, Object]
----

`kafkaParamsForExecutors` sets the <<kafkaParamsForExecutors-properties, Kafka properties for executors>>.

While setting the properties, `kafkaParamsForExecutors` prints out the following DEBUG message to the logs:

```
executor: Set [key] to [value], earlier value: [value]
```

[NOTE]
====
`kafkaParamsForExecutors` is used when:

* `KafkaSourceProvider` is requested to <<createSource, createSource>> (for a <<spark-sql-streaming-KafkaSource.md#, KafkaSource>>), <<createMicroBatchReader, createMicroBatchReader>> (for a <<spark-sql-streaming-KafkaMicroBatchReader.md#, KafkaMicroBatchReader>>), and <<createContinuousReader, createContinuousReader>> (for a <<spark-sql-streaming-KafkaContinuousReader.md#, KafkaContinuousReader>>)

* `KafkaRelation` is requested to <<spark-sql-streaming-KafkaRelation.md#buildScan, buildScan>> (for a `KafkaSourceRDD`)
====

=== [[failOnDataLoss]] Looking Up failOnDataLoss Configuration Property -- `failOnDataLoss` Internal Method

[source, scala]
----
failOnDataLoss(caseInsensitiveParams: Map[String, String]): Boolean
----

`failOnDataLoss` simply looks up the `failOnDataLoss` configuration property in the given `caseInsensitiveParams` (in case-insensitive manner) or defaults to `true`.

NOTE: `failOnDataLoss` is used when `KafkaSourceProvider` is requested to <<createSource, createSource>> (for a <<spark-sql-streaming-KafkaSource.md#, KafkaSource>>), <<createMicroBatchReader, createMicroBatchReader>> (for a <<spark-sql-streaming-KafkaMicroBatchReader.md#, KafkaMicroBatchReader>>), <<createContinuousReader, createContinuousReader>> (for a <<spark-sql-streaming-KafkaContinuousReader.md#, KafkaContinuousReader>>), and <<createRelation, createRelation>> (for a <<spark-sql-streaming-KafkaRelation.md#, KafkaRelation>>).
