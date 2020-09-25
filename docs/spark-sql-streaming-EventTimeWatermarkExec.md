== [[EventTimeWatermarkExec]] EventTimeWatermarkExec Unary Physical Operator

`EventTimeWatermarkExec` is a unary physical operator that represents <<spark-sql-streaming-EventTimeWatermark.md#, EventTimeWatermark>> logical operator at execution time.

[NOTE]
====
A unary physical operator (`UnaryExecNode`) is a physical operator with a single <<child, child>> physical operator.

Read up on https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-SparkPlan.html[UnaryExecNode] (and physical operators in general) in https://bit.ly/spark-sql-internals[The Internals of Spark SQL] book.
====

The <<doExecute, purpose>> of the `EventTimeWatermarkExec` operator is to simply extract (_project_) the values of the <<eventTime, event-time watermark column>> and add them directly to the <<eventTimeStats, EventTimeStatsAccum>> internal accumulator.

[NOTE]
====
Since the execution (data processing) happens on Spark executors, the only way to establish communication between the tasks (on the executors) and the driver is to use an accumulator.

Read up on https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-accumulators.html[Accumulators] in https://bit.ly/apache-spark-internals[The Internals of Apache Spark] book.
====

`EventTimeWatermarkExec` uses <<eventTimeStats, EventTimeStatsAccum>> internal accumulator as a way to send the statistics (the maximum, minimum, average and update count) of the values in the <<eventTime, event-time watermark column>> that is later used in:

* `ProgressReporter` for [creating execution statistics](ProgressReporter.md#extractExecutionStats) for the most recent query execution (for monitoring the `max`, `min`, `avg`, and `watermark` event-time watermark statistics)

* `StreamExecution` to observe and possibly update event-time watermark when <<spark-sql-streaming-MicroBatchExecution.md#constructNextBatch-hasNewData-true, constructing the next streaming batch>>.

`EventTimeWatermarkExec` is <<creating-instance, created>> exclusively when <<spark-sql-streaming-StatefulAggregationStrategy.md#, StatefulAggregationStrategy>> execution planning strategy is requested to plan a logical plan with <<spark-sql-streaming-EventTimeWatermark.md#, EventTimeWatermark>> logical operators for execution.

TIP: Check out <<spark-sql-streaming-demo-watermark-aggregation-append.md#, Demo: Streaming Watermark with Aggregation in Append Output Mode>> to deep dive into the internals of <<spark-sql-streaming-watermark.md#, Streaming Watermark>>.

=== [[creating-instance]] Creating EventTimeWatermarkExec Instance

`EventTimeWatermarkExec` takes the following to be created:

* [[eventTime]] *Event time column* - the column with the (event) time for event-time watermark
* [[delay]] Delay interval (`CalendarInterval`)
* [[child]] Child physical operator (`SparkPlan`)

While <<creating-instance, being created>>, `EventTimeWatermarkExec` registers the <<eventTimeStats, EventTimeStatsAccum>> internal accumulator (with the current `SparkContext`).

=== [[doExecute]] Executing Physical Operator (Generating RDD[InternalRow]) -- `doExecute` Method

[source, scala]
----
doExecute(): RDD[InternalRow]
----

NOTE: `doExecute` is part of `SparkPlan` Contract to generate the runtime representation of an physical operator as a distributed computation over internal binary rows on Apache Spark (i.e. `RDD[InternalRow]`).

Internally, `doExecute` executes the <<child, child>> physical operator and maps over the partitions (using `RDD.mapPartitions`).

`doExecute` creates an unsafe projection (one per partition) for the <<eventTime, column with the event time>> in the output schema of the <<child, child>> physical operator. The unsafe projection is to extract event times from the (stream of) internal rows of the child physical operator.

For every row (`InternalRow`) per partition, `doExecute` requests the <<eventTimeStats, eventTimeStats>> accumulator to <<spark-sql-streaming-EventTimeStatsAccum.md#add, add the event time>>.

NOTE: The event time value is in seconds (not millis as the value is divided by `1000` ).

=== [[output]] Output Attributes (Schema) -- `output` Property

[source, scala]
----
output: Seq[Attribute]
----

NOTE: `output` is part of the `QueryPlan` Contract to describe the attributes of (the schema of) the output.

`output` requests the <<child, child>> physical operator for the output attributes to find the <<eventTime, event time column>> and any other column with metadata that contains <<spark-sql-streaming-EventTimeWatermark.md#delayKey, spark.watermarkDelayMs>> key.

For the <<eventTime, event time column>>, `output` updates the metadata to include the <<delayMs, delay interval>> for the <<spark-sql-streaming-EventTimeWatermark.md#delayKey, spark.watermarkDelayMs>> key.

For any other column (not the <<eventTime, event time column>>) with the <<spark-sql-streaming-EventTimeWatermark.md#delayKey, spark.watermarkDelayMs>> key, `output` simply removes the key from the metadata.

[source, scala]
----
// FIXME: Would be nice to have a demo. Anyone?
----

=== [[internal-properties]] Internal Properties

[cols="30m,70",options="header",width="100%"]
|===
| Name
| Description

| delayMs
a| [[delayMs]] *Delay interval* - the <<delay, delay>> interval in milliseconds

Used when:

* `EventTimeWatermarkExec` is requested for the <<output, output attributes>>

* `WatermarkTracker` is requested to <<spark-sql-streaming-WatermarkTracker.md#updateWatermark, update the event-time watermark>>

| eventTimeStats
a| [[eventTimeStats]] <<spark-sql-streaming-EventTimeStatsAccum.md#, EventTimeStatsAccum>> accumulator to accumulate <<eventTime, eventTime>> values from every row in a streaming batch (when `EventTimeWatermarkExec` <<doExecute, is executed>>).

NOTE: `EventTimeStatsAccum` is a Spark accumulator of `EventTimeStats` from `Longs` (i.e. `AccumulatorV2[Long, EventTimeStats]`).

NOTE: Every Spark accumulator has to be registered before use, and `eventTimeStats` is registered when `EventTimeWatermarkExec` <<creating-instance, is created>>.

|===
