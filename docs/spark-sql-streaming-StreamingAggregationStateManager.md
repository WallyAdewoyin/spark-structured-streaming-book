== [[StreamingAggregationStateManager]] StreamingAggregationStateManager Contract -- State Managers for Streaming Aggregation

`StreamingAggregationStateManager` is the <<contract, abstraction>> of <<implementations, state managers>> that act as _middlemen_ between <<spark-sql-streaming-StateStore.md#, state stores>> and the physical operators used in <<spark-sql-streaming-aggregation.md#, Streaming Aggregation>> (e.g. <<StateStoreSaveExec.md#, StateStoreSaveExec>> and <<spark-sql-streaming-StateStoreRestoreExec.md#, StateStoreRestoreExec>>).

[[contract]]
.StreamingAggregationStateManager Contract
[cols="1m,2",options="header",width="100%"]
|===
| Method
| Description

| commit
a| [[commit]]

[source, scala]
----
commit(
  store: StateStore): Long
----

Commits all updates (_changes_) to the given <<spark-sql-streaming-StateStore.md#, state store>> and returns the new version

Used exclusively when <<StateStoreSaveExec.md#, StateStoreSaveExec>> physical operator is executed.

| get
a| [[get]]

[source, scala]
----
get(store: StateStore, key: UnsafeRow): UnsafeRow
----

Looks up the value of the key from the <<spark-sql-streaming-StateStore.md#, state store>> (the key is non-``null``)

Used exclusively when <<spark-sql-streaming-StateStoreRestoreExec.md#, StateStoreRestoreExec>> physical operator is executed.

| getKey
a| [[getKey]]

[source, scala]
----
getKey(row: UnsafeRow): UnsafeRow
----

Extracts the columns for the key from the input row

Used when:

* <<spark-sql-streaming-StateStoreRestoreExec.md#, StateStoreRestoreExec>> physical operator is executed

* `StreamingAggregationStateManagerImplV1` legacy state manager is requested to <<spark-sql-streaming-StreamingAggregationStateManagerImplV1.md#put, put a row to a state store>>

| getStateValueSchema
a| [[getStateValueSchema]]

[source, scala]
----
getStateValueSchema: StructType
----

Gets the schema of the values in a <<spark-sql-streaming-StateStore.md#, state store>>

Used when <<spark-sql-streaming-StateStoreRestoreExec.md#, StateStoreRestoreExec>> and <<StateStoreSaveExec.md#, StateStoreSaveExec>> physical operators are executed

| iterator
a| [[iterator]]

[source, scala]
----
iterator(
  store: StateStore): Iterator[UnsafeRowPair]
----

Returns all `UnsafeRow` key-value pairs in the given <<spark-sql-streaming-StateStore.md#, state store>>

Used exclusively when <<StateStoreSaveExec.md#, StateStoreSaveExec>> physical operator is executed.

| keys
a| [[keys]]

[source, scala]
----
keys(store: StateStore): Iterator[UnsafeRow]
----

Returns all the keys in the <<spark-sql-streaming-StateStore.md#, state store>>

Used exclusively when physical operators with `WatermarkSupport` are requested to <<spark-sql-streaming-WatermarkSupport.md#removeKeysOlderThanWatermark-StreamingAggregationStateManager-store, removeKeysOlderThanWatermark>> (i.e. exclusively when <<StateStoreSaveExec.md#, StateStoreSaveExec>> physical operator is executed).

| put
a| [[put]]

[source, scala]
----
put(
  store: StateStore,
  row: UnsafeRow): Unit
----

Stores (_puts_) the given row in the given <<spark-sql-streaming-StateStore.md#, state store>>

Used exclusively when <<StateStoreSaveExec.md#, StateStoreSaveExec>> physical operator is executed.

| remove
a| [[remove]]

[source, scala]
----
remove(
  store: StateStore,
  key: UnsafeRow): Unit
----

Removes the key-value pair from the given <<spark-sql-streaming-StateStore.md#, state store>> per key

Used exclusively when <<StateStoreSaveExec.md#, StateStoreSaveExec>> physical operator is executed (directly or indirectly as a <<spark-sql-streaming-WatermarkSupport.md#removeKeysOlderThanWatermark-StreamingAggregationStateManager-store, WatermarkSupport>>)

| values
a| [[values]]

[source, scala]
----
values(
  store: StateStore): Iterator[UnsafeRow]
----

All values in the <<spark-sql-streaming-StateStore.md#, state store>>

Used exclusively when <<StateStoreSaveExec.md#, StateStoreSaveExec>> physical operator is executed.

|===

[[supportedVersions]]
`StreamingAggregationStateManager` supports <<createStateManager, two versions of state managers for streaming aggregations>> (per the <<spark-sql-streaming-properties.md#spark.sql.streaming.aggregation.stateFormatVersion, spark.sql.streaming.aggregation.stateFormatVersion>> internal configuration property):

* [[legacyVersion]] `1` (for the legacy <<spark-sql-streaming-StreamingAggregationStateManagerBaseImpl.md#StreamingAggregationStateManagerImplV1, StreamingAggregationStateManagerImplV1>>)

* [[default]] `2` (for the default <<spark-sql-streaming-StreamingAggregationStateManagerBaseImpl.md#StreamingAggregationStateManagerImplV2, StreamingAggregationStateManagerImplV2>>)

[[implementations]]
NOTE: <<spark-sql-streaming-StreamingAggregationStateManagerBaseImpl.md#, StreamingAggregationStateManagerBaseImpl>> is the one and only known direct implementation of the <<contract, StreamingAggregationStateManager Contract>> in Spark Structured Streaming.

NOTE: `StreamingAggregationStateManager` is a Scala *sealed trait* which means that all the <<implementations, implementations>> are in the same compilation unit (a single file).

=== [[createStateManager]] Creating StreamingAggregationStateManager Instance -- `createStateManager` Factory Method

[source, scala]
----
createStateManager(
  keyExpressions: Seq[Attribute],
  inputRowAttributes: Seq[Attribute],
  stateFormatVersion: Int): StreamingAggregationStateManager
----

`createStateManager` creates a new `StreamingAggregationStateManager` for a given `stateFormatVersion`:

* <<spark-sql-streaming-StreamingAggregationStateManagerImplV1.md#, StreamingAggregationStateManagerImplV1>> for `stateFormatVersion` being `1`

* <<spark-sql-streaming-StreamingAggregationStateManagerImplV2.md#, StreamingAggregationStateManagerImplV2>> for `stateFormatVersion` being `2`

`createStateManager` throws a `IllegalArgumentException` for any other `stateFormatVersion`:

```
Version [stateFormatVersion] is invalid
```

NOTE: `createStateManager` is used when <<spark-sql-streaming-StateStoreRestoreExec.md#stateManager, StateStoreRestoreExec>> and <<StateStoreSaveExec.md#stateManager, StateStoreSaveExec>> physical operators are created.
