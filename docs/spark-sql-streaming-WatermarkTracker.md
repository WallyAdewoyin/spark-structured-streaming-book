== [[WatermarkTracker]] WatermarkTracker

`WatermarkTracker` tracks the <<globalWatermarkMs, event-time watermark>> of a streaming query (across <<operatorToWatermarkMap, EventTimeWatermarkExec operators>> in a physical query plan) based on a given <<policy, MultipleWatermarkPolicy>>.

`WatermarkTracker` is used exclusively in <<MicroBatchExecution.md#watermarkTracker, MicroBatchExecution>>.

`WatermarkTracker` is <<creating-instance, created>> (using the <<apply, factory method>>) when `MicroBatchExecution` is requested to <<MicroBatchExecution.md#populateStartOffsets, populate start offsets>> (when requested to <<MicroBatchExecution.md#runActivatedStream, run an activated streaming query>>).

[[policy]]
[[creating-instance]]
`WatermarkTracker` takes a single <<MultipleWatermarkPolicy, MultipleWatermarkPolicy>> to be created.

[[MultipleWatermarkPolicy]]
`MultipleWatermarkPolicy` can be one of the following:

* [[MaxWatermark]] `MaxWatermark` (alias: `min`)
* [[MinWatermark]] `MinWatermark` (alias: `max`)

[[logging]]
[TIP]
====
Enable `ALL` logging level for `org.apache.spark.sql.execution.streaming.WatermarkTracker` to see what happens inside.

Add the following line to `conf/log4j.properties`:

```
log4j.logger.org.apache.spark.sql.execution.streaming.WatermarkTracker=ALL
```

Refer to <<spark-sql-streaming-spark-logging.md#, Logging>>.
====

=== [[apply]] Creating WatermarkTracker -- `apply` Factory Method

[source, scala]
----
apply(conf: RuntimeConfig): WatermarkTracker
----

`apply` uses the <<spark-sql-streaming-properties.md#spark.sql.streaming.multipleWatermarkPolicy, spark.sql.streaming.multipleWatermarkPolicy>> configuration property for the global watermark policy (default: `min`) and creates a <<creating-instance, WatermarkTracker>>.

NOTE: `apply` is used exclusively when `MicroBatchExecution` is requested to <<MicroBatchExecution.md#populateStartOffsets, populate start offsets>> (when requested to <<MicroBatchExecution.md#runActivatedStream, run an activated streaming query>>).

=== [[setWatermark]] `setWatermark` Method

[source, scala]
----
setWatermark(newWatermarkMs: Long): Unit
----

`setWatermark` simply updates the <<globalwatermarkms, global event-time watermark>> to the given `newWatermarkMs`.

NOTE: `setWatermark` is used exclusively when `MicroBatchExecution` is requested to <<MicroBatchExecution.md#populateStartOffsets, populate start offsets>> (when requested to <<MicroBatchExecution.md#runActivatedStream, run an activated streaming query>>).

=== [[updateWatermark]] Updating Event-Time Watermark -- `updateWatermark` Method

[source, scala]
----
updateWatermark(executedPlan: SparkPlan): Unit
----

`updateWatermark` requests the given physical operator (`SparkPlan`) to collect all [EventTimeWatermarkExec](physical-operators/EventTimeWatermarkExec.md) unary physical operators.

`updateWatermark` simply exits when no `EventTimeWatermarkExec` was found.

`updateWatermark`...FIXME

NOTE: `updateWatermark` is used exclusively when `MicroBatchExecution` is requested to <<MicroBatchExecution.md#runBatch, run a single streaming batch>> (when requested to <<MicroBatchExecution.md#runActivatedStream, run an activated streaming query>>).

=== [[internal-properties]] Internal Properties

[cols="30m,70",options="header",width="100%"]
|===
| Name
| Description

| globalWatermarkMs
a| [[globalWatermarkMs]][[currentWatermark]] Current *global event-time watermark* per <<policy, MultipleWatermarkPolicy>> (across <<operatorToWatermarkMap, all EventTimeWatermarkExec operators>> in a physical query plan)

Default: `0`

Used when...FIXME

| operatorToWatermarkMap
a| [[operatorToWatermarkMap]] Event-time watermark per [EventTimeWatermarkExec](physical-operators/EventTimeWatermarkExec.md) physical operator (`mutable.HashMap[Int, Long]`)

Used when...FIXME

|===
