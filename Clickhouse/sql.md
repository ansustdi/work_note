### move partition 
```sql 
ALTER TABLE bcom_vbal  
    MOVE PARTITION '202001' TO TABLE bcom_vbal_hist;
```
### get partition script
```sql 
SELECT  
    'ALTER TABLE bcom_vbal MOVE PARTITION ''' || partition || ''' TO TABLE bcom_vbal_hist;'FROM system.parts  
WHERE table = 'bcom_vbal'  
  AND active  
  AND partition >= '199801'  
  AND partition <= '202412'  
GROUP BY partition  
ORDER BY partition;
```

### bcom_vbal  check query 
```sql 
SELECT acnt_code, bal_type_code, offbal, cycle_no, started, count()  
FROM  
    bcom_vbal  
WHERE  
    ended = '2149-06-06'  
GROUP BY  
    acnt_code, bal_type_code, offbal, cycle_no, started  
HAVING  
    count() > 1;
```

### Kill mutation 
```sql 
KILL MUTATION WHERE table = 'loan_acnt_fact' AND mutation_id = 'mutation_9240.txt';
```

### delete rows by partition
```sql 
ALTER  
TABLE  
wh.loan_acnt_fact  
DELETE  
WHERE  
    toYYYYMM(bal_date) = 201803  
    AND (acnt_code, bal_date) IN (SELECT f.acnt_code, f.bal_date  
    FROM  
        wh.loan_acnt_fact f  
            INNER JOIN wh.loan_acnt a ON f.acnt_code = a.acnt_code  
    WHERE  
        toYYYYMM(f.bal_date) = 201803  
        AND a.status = 'C'  
        AND a.closed_date IS NOT NULL  
        AND f.bal_date > a.closed_date);
```

### python script that delete rows by partition automatically 
```python
from clickhouse_driver import Client
import time
from datetime import date


client = Client(
    host="172.16.140.90",
    database="wh",
    user="dwh",
    password="axK0b15dP4e9WO9",
)


def generate_partitions(start_ym: int, end_ym: int):
    """Generate partition list from YYYYMM to YYYYMM inclusive."""
    partitions = []
    y, m = divmod(start_ym, 100)

    while True:
        partitions.append(y * 100 + m)

        if y * 100 + m == end_ym:
            break

        m += 1
        if m > 12:
            m = 1
            y += 1

    return partitions


def wait_for_mutations(table="loan_acnt_fact", poll_interval=5):
    """Wait until all mutations on the target table are finished."""
    while True:
        rows = client.execute(
            f"""
            SELECT count()
            FROM system.mutations
            WHERE table = '{table}'
              AND is_done = 0
            """
        )
        pending = rows[0][0]

        if pending == 0:
            break

        print(f"  Waiting... {pending} mutation(s) still running", flush=True)
        time.sleep(poll_interval)


def check_last_fail(table="loan_acnt_fact"):
    """Check latest mutation failure for the target table."""
    rows = client.execute(
        f"""
        SELECT mutation_id, latest_fail_reason
        FROM system.mutations
        WHERE table = '{table}'
          AND latest_fail_reason != ''
        ORDER BY create_time DESC
        LIMIT 1
        """
    )

    if rows and rows[0][1]:
        return rows[0]

    return None


# -------------------------------------------------------
# Start from 201805 since 201801–201804 were already run
# Change START_YM if resuming from a different point
# -------------------------------------------------------
START_YM = 201805
END_YM = 202603

partitions = generate_partitions(START_YM, END_YM)
total = len(partitions)

print(f"Total partitions to process: {total}")

for i, ym in enumerate(partitions, 1):
    print(f"\n[{i}/{total}] Processing partition {ym}...", flush=True)

    client.execute(
        f"""
        ALTER TABLE wh.loan_acnt_fact DELETE
        WHERE toYYYYMM(bal_date) = {ym}
          AND (acnt_code, bal_date) IN (
              SELECT f.acnt_code, f.bal_date
              FROM wh.loan_acnt_fact f
              INNER JOIN wh.loan_acnt a
                  ON f.acnt_code = a.acnt_code
              WHERE toYYYYMM(f.bal_date) = {ym}
                AND a.status = 'C'
                AND a.closed_date IS NOT NULL
                AND f.bal_date > a.closed_date
          )
        """
    )

    wait_for_mutations()

    fail = check_last_fail()
    if fail:
        print(f"\n❌ Mutation failed on partition {ym}!")
        print(f"   mutation_id: {fail[0]}")
        print(f"   reason: {fail[1]}")
        print(f"   Stopping. Fix the issue and restart from START_YM = {ym}")
        break

    print(f"  ✅ Partition {ym} done.")

print("\nAll partitions processed.")
```

### see monthly data size 
```sql 
SELECT  
    toStartOfMonth(modification_time) AS month,  
    formatReadableSize(sum(bytes_on_disk)) AS size_added,  
    round(sum(bytes_on_disk) / 1073741824, 2) AS gb_added  
FROM system.parts  
WHERE active = 1  
GROUP BY month  
ORDER BY month;
```

### see total size 
```sql 
SELECT  
    formatReadableSize(sum(bytes_on_disk)) AS total_on_disk,  
    formatReadableSize(sum(data_uncompressed_bytes)) AS total_uncompressed,  
    round(sum(data_uncompressed_bytes) / sum(bytes_on_disk), 2) AS compression_ratio  
FROM system.parts  
WHERE active = 1;
```

### Compare per-table compression in each database
```sql 
SELECT  
    database,  
    table,  
    formatReadableSize(sum(bytes_on_disk)) AS on_disk,  
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,  
    round(sum(data_uncompressed_bytes) / sum(bytes_on_disk), 2) AS ratio  
FROM  
    system.parts  
WHERE  
    active = 1  
GROUP BY  
    database, table  
ORDER BY  
    database, ratio DESC;
```

###  Set TTL on system tables to reclaim disk automatically
```sql 
ALTER TABLE system.trace_log MODIFY TTL event_time + INTERVAL 30 DAY;  
ALTER TABLE system.asynchronous_metric_log MODIFY TTL event_time + INTERVAL 7 DAY;  
ALTER TABLE system.text_log MODIFY TTL event_time + INTERVAL 7 DAY;
```

```sql 
  
CREATE TABLE lake.lake_mg_cust_log_arch_v2  
(  
    log_id         Int64 CODEC (Delta, ZSTD(3)),  
    log_datetime   DateTime CODEC (DoubleDelta, ZSTD(3)),  
  
    -- High cardinality strings: ZSTD only  
    cust_id        Nullable(String) CODEC (ZSTD(3)),  
    session_id     Nullable(String) CODEC (ZSTD(3)),  
    wallet         Nullable(String) CODEC (ZSTD(3)),  
    device_id      Nullable(String) CODEC (ZSTD(3)),  
    mac            Nullable(String) CODEC (ZSTD(3)),  
    ip             Nullable(String) CODEC (ZSTD(3)),  
    detail_info    Nullable(String) CODEC (ZSTD(6)), -- likely biggest  
    log_option     Nullable(String) CODEC (ZSTD(6)),  
    res_desc       Nullable(String) CODEC (ZSTD(6)),  
    os_version     Nullable(String) CODEC (ZSTD(3)),  
    device_name    Nullable(String) CODEC (ZSTD(3)),  
    country        Nullable(String) CODEC (ZSTD(3)),  
    city           Nullable(String) CODEC (ZSTD(3)),  
    district       Nullable(String) CODEC (ZSTD(3)),  
    street         Nullable(String) CODEC (ZSTD(3)),  
  
    -- Low cardinality enums: benefit most from ZSTD  
    chnl_type      LowCardinality(Nullable(String)) CODEC (ZSTD(3)),  
    status         LowCardinality(Nullable(String)) CODEC (ZSTD(3)),  
    log_type_code  LowCardinality(Nullable(String)) CODEC (ZSTD(3)),  
    device_status  LowCardinality(Nullable(String)) CODEC (ZSTD(3)),  
    res_code       LowCardinality(Nullable(String)) CODEC (ZSTD(3)),  
    cust_status    LowCardinality(Nullable(String)) CODEC (ZSTD(3)),  
    entity_code    LowCardinality(Nullable(String)) CODEC (ZSTD(3)),  
  
    -- GPS: store as Float32 instead of Decimal(10,6) — saves 4 bytes per row per column  
    longitude      Nullable(Float32) CODEC (Gorilla, ZSTD(3)),  
    latitude       Nullable(Float32) CODEC (Gorilla, ZSTD(3)),  
  
    -- Integers  
    app_version    Nullable(Int64) CODEC (Delta, ZSTD(3)),  
    thread_id      Nullable(Int64) CODEC (Delta, ZSTD(3)),  
    app_id         Nullable(Int32) CODEC (Delta, ZSTD(3)),  
    user_id        Nullable(Int32) CODEC (Delta, ZSTD(3)),  
    with_fp        Nullable(Int8) CODEC (ZSTD(3)),  
  
    -- Datetime columns  
    start_datetime Nullable(DateTime) CODEC (ZSTD(3)),  
    end_datetime   Nullable(DateTime) CODEC (ZSTD(3)),  
    updated_at     DateTime DEFAULT now() CODEC (DoubleDelta, ZSTD(3))  
)  
    ENGINE = ReplacingMergeTree(updated_at)  
-- Partition by month: enables TTL, partition drops, pruning  
        PARTITION BY toYYYYMM(log_datetime)  
        ORDER BY (log_id)  
        SETTINGS index_granularity = 8192;
```
### see table size 
```sql 
SELECT  
    table,  
    formatReadableSize(sum(bytes_on_disk)) AS on_disk,  
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,  
    round(sum(data_uncompressed_bytes) / sum(bytes_on_disk), 2) AS ratio  
FROM  
    system.parts  
WHERE  
    database = 'lake'  
    AND table IN ('lake_mg_cust_log', 'lake_mg_cust_log_v2')  
    AND active = 1  
GROUP BY  
    table;
```
### check refresh 
```sql 
SELECT *  
FROM  
    system.view_refreshes  
WHERE  
    view = 'mv_lending';
```


### see size by database 
```sql 
SELECT  
    database,  
    formatReadableSize(sum(bytes_on_disk)) AS on_disk,  
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,  
    round(sum(data_uncompressed_bytes) / sum(bytes_on_disk), 2) AS ratio  
FROM  
    system.parts  
WHERE  
    active = 1  
GROUP BY  
    database  
ORDER BY  
    ratio DESC;
```