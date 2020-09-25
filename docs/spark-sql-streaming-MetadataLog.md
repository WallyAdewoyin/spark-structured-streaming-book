== [[MetadataLog]] MetadataLog Contract -- Metadata Storage

`MetadataLog` is the <<contract, abstraction>> of <<implementations, metadata storage>> that can <<add, persist>>, <<get, retrieve>>, and <<purge, remove>> metadata (of type `T`).

[[contract]]
.MetadataLog Contract
[cols="30m,70",options="header",width="100%"]
|===
| Method
| Description

| add
a| [[add]]

[source, scala]
----
add(
  batchId: Long,
  metadata: T): Boolean
----

Persists (_adds_) metadata of a streaming batch

Used when:

* `KafkaMicroBatchReader` is requested to <<spark-sql-streaming-KafkaMicroBatchReader.md#getOrCreateInitialPartitionOffsets, getOrCreateInitialPartitionOffsets>>

* `KafkaSource` is requested for the <<spark-sql-streaming-KafkaSource.md#initialPartitionOffsets, initialPartitionOffsets>>

* `CompactibleFileStreamLog` is requested for the <<spark-sql-streaming-CompactibleFileStreamLog.md#add, store metadata of a streaming batch>> and to <<spark-sql-streaming-CompactibleFileStreamLog.md#compact, compact>>

* `FileStreamSource` is requested to <<spark-sql-streaming-FileStreamSource.md#fetchMaxOffset, fetchMaxOffset>>

* `FileStreamSourceLog` is requested to <<spark-sql-streaming-FileStreamSourceLog.md#add, store (add) metadata of a streaming batch>>

* `ManifestFileCommitProtocol` is requested to <<spark-sql-streaming-ManifestFileCommitProtocol.md#commitJob, commitJob>>

* `MicroBatchExecution` stream execution engine is requested to <<MicroBatchExecution.md#constructNextBatch, construct a next streaming micro-batch>> and <<MicroBatchExecution.md#runBatch, run a single streaming micro-batch>>

* `ContinuousExecution` stream execution engine is requested to <<ContinuousExecution.md#addOffset, addOffset>> and <<ContinuousExecution.md#commit, commit an epoch>>

* `RateStreamMicroBatchReader` is created (`creationTimeMs`)

| get
a| [[get]]

[source, scala]
----
get(
  batchId: Long): Option[T]
get(
  startId: Option[Long],
  endId: Option[Long]): Array[(Long, T)]
----

Retrieves (_gets_) metadata of one or more batches

Used when...FIXME

| getLatest
a| [[getLatest]]

[source, scala]
----
getLatest(): Option[(Long, T)]
----

Retrieves the latest-committed metadata (if available)

Used when...FIXME

| purge
a| [[purge]]

[source, scala]
----
purge(thresholdBatchId: Long): Unit
----

Used when...FIXME

|===

[[implementations]]
NOTE: <<spark-sql-streaming-HDFSMetadataLog.md#, HDFSMetadataLog>> is the only direct implementation of the <<contract, MetadataLog Contract>> in Spark Structured Streaming.
