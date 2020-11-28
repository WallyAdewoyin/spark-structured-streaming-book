# StateStoreMetrics

[[creating-instance]]
`StateStoreMetrics` holds the performance metrics of a <<spark-sql-streaming-StateStore.md#, state store>>:

* [[numKeys]] Number of keys
* [[memoryUsedBytes]] Memory used (in bytes)
* [[customMetrics]] <<spark-sql-streaming-StateStoreCustomMetric.md#, StateStoreCustomMetrics>> with their current values (`Map[StateStoreCustomMetric, Long]`)

`StateStoreMetrics` is used (and <<creating-instance, created>>) when the following are requested for the performance metrics:

* [StateStore](spark-sql-streaming-StateStore.md#metrics)

* [StateStoreHandler](spark-sql-streaming-StateStoreHandler.md#metrics)

* [SymmetricHashJoinStateManager](SymmetricHashJoinStateManager.md#metrics)
