ReplicatedMergeTree：MergeTree的副本版本引擎

```sql
CREATE TABLE develop_test_local ON CLUSTER 'KC0_CK_TS_01'
(
CounterID int,
Date Datetime
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/KC0_CK_TS_01/jdob_ha/default/develop_test_local/{shard}', '{replica}')
PARTITION BY toYYYYMM(Date)
ORDER BY (CounterID,Date)
SETTINGS storage_policy = 'jdob_ha', index_granularity=8192;
```


