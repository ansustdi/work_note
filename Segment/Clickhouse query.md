### Create Table Script
```sql 
CREATE TABLE acnt_seg.seg_txn  
(  
    `cust_code` String,  
    `acnt_code` String,  
    `txn_datetime`  DateTime,  
    `txn_amt`   Decimal(38, 10),  
    `is_credit` UInt8, -- 1 for credit, 0 for debit  
    `oper_code` String  
)  
    ENGINE = MergeTree()  
        PARTITION BY toYYYYMM(txn_datetime)  
        ORDER BY (cust_code, acnt_code, txn_datetime);
```

```sql 
CREATE TABLE acnt_seg.seg_txn_aggr_metrics
(
    `cust_code`          String,
    `acnt_code`          String,
    `period`             Enum8('1m' = 1, '3m' = 2, '6m' = 3, '1y' = 4, 'all' = 5),
    `oper_code`          String,
    `credit_txn_amt_sum` AggregateFunction(sumIf, Decimal(38, 10), UInt8),
    `debit_txn_amt_sum`  AggregateFunction(sumIf, Decimal(38, 10), UInt8),
    `credit_txn_count`   AggregateFunction(countIf, UInt8),
    `debit_txn_count`    AggregateFunction(countIf, UInt8),

    `last_updated`       DateTime DEFAULT now()
)
ENGINE = AggregatingMergeTree()
ORDER BY (cust_code, acnt_code, oper_code, period);
```

```sql
CREATE MATERIALIZED VIEW acnt_seg.mv_seg_metrics TO acnt_seg.seg_txn_aggr_metrics  
AS  
WITH  
    periods AS (SELECT  
        cust_code,  
        acnt_code,  
        txn_datetime,  
        txn_amt,  
        is_credit,  
        oper_code,  
        arrayJoin(['1m', '3m', '6m', '1y', 'all']) AS period  
    FROM  
        acnt_seg.seg_txn)  
SELECT  
    cust_code,  
    acnt_code,  
    period,  
    oper_code,  
    sumIfState(txn_amt, is_credit = 1) AS credit_txn_amt_sum,  
    sumIfState(txn_amt, is_credit = 0) AS debit_txn_amt_sum,  
    countIfState(is_credit = 1) AS credit_txn_count,  
    countIfState(is_credit = 0) AS debit_txn_count  
FROM  
    periods  
WHERE  
    (period = '1m' AND txn_datetime >= now() - INTERVAL 1 MONTH)  
    OR (period = '3m' AND txn_datetime >= now() - INTERVAL 3 MONTH)  
    OR (period = '6m' AND txn_datetime >= now() - INTERVAL 6 MONTH)  
    OR (period = '1y' AND txn_datetime >= now() - INTERVAL 1 YEAR)  
    OR (period = 'all')  
GROUP BY  
    cust_code,  
    acnt_code,  
    oper_code,  
    period;
```

### select sql 
```sql 
WITH  
    filtered_casa_txn AS (SELECT  
        acnt_code,  
        txn_amount * currate AS txn_amount, post_date, cont_acnt_code, txn_type  
    FROM  
        wh.casa_fintxn  
    WHERE  
        txn_date = '2026-01-21'  
        AND corr = 0  
        AND txn_type IN ('CR', 'DR')  
        AND corr = 0  
        AND cur_code NOT IN ('ACO', 'RDX')  
        AND cont_acnt_code IS NOT NULL),  
    filtered_cust AS (SELECT acnt_code, cust_code  
    FROM  
        wh.bcom_acnt)  
SELECT  
    c.cust_code AS cust_code,  
    main.acnt_code AS acnt_code,  
    main.post_date AS txn_datetime,  
    main.txn_amount AS txn_amt,  
    if(main.txn_type = 'CR', 1, 0) AS is_credit,  
    t.OPER_CODE AS oper_code  
FROM  
    filtered_casa_txn main  
        JOIN filtered_cust c ON c.acnt_code = main.acnt_code  
        JOIN acnt_seg.SEG_ACNT_CAT t ON t.ACNT_CODE = main.cont_acnt_code;
```

### second approach batch based 
```sql 
-- Use mutations_sync to make the process wait for the delete to complete
ALTER TABLE acnt_seg.seg_txn
DELETE WHERE toDate(txn_datetime) = '2026-01-21'
SETTINGS mutations_sync = 2;
```

```sql 
-- Insert the fresh data for the EOD date
INSERT INTO acnt_seg.seg_txn (cust_code, acnt_code, txn_datetime, txn_amt, is_credit, oper_code)
WITH
    filtered_casa_txn AS (SELECT
        acnt_code,
        txn_amount * currate AS txn_amount, post_date, cont_acnt_code, txn_type
    FROM
        wh.casa_fintxn
    WHERE
        toDate(txn_date) = '2026-01-21' -- Safer date check
        AND corr = 0
        AND txn_type IN ('CR', 'DR')
        AND cur_code NOT IN ('ACO', 'RDX')
        AND cont_acnt_code IS NOT NULL),
    filtered_cust AS (SELECT acnt_code, cust_code
    FROM
        wh.bcom_acnt)
SELECT
    c.cust_code AS cust_code,
    main.acnt_code AS acnt_code,
    main.post_date AS txn_datetime,
    main.txn_amount AS txn_amt,
    if(main.txn_type = 'CR', 1, 0) AS is_credit,
    t.OPER_CODE AS oper_code
FROM
    filtered_casa_txn AS main
        JOIN filtered_cust AS c ON c.acnt_code = main.acnt_code
        JOIN acnt_seg.SEG_ACNT_CAT AS t ON t.ACNT_CODE = main.cont_acnt_code;
```

```sql 
TRUNCATE TABLE acnt_seg.seg_txn_aggr_metrics;
```

```sql 
-- Rebuild the entire aggregated table from the full history in seg_txn
INSERT INTO acnt_seg.seg_txn_aggr_metrics
WITH
    periods AS (SELECT
        cust_code,
        acnt_code,
        txn_datetime,
        txn_amt,
        is_credit,
        oper_code,
        arrayJoin(['1m', '3m', '6m', '1y', 'all']) AS period
    FROM
        acnt_seg.seg_txn -- Reading from our FAST ClickHouse staging table
    )
SELECT
    cust_code,
    acnt_code,
    period,
    oper_code,
    sumIfState(txn_amt, is_credit = 1), -- AS credit_txn_amt_sum
    sumIfState(txn_amt, is_credit = 0), -- AS debit_txn_amt_sum
    countIfState(is_credit = 1),       -- AS credit_txn_count
    countIfState(is_credit = 0)        -- AS debit_txn_count
FROM
    periods
WHERE
    -- This logic is correct for recalculating all rolling periods
    (period = '1m' AND txn_datetime >= now() - INTERVAL 1 MONTH)
    OR (period = '3m' AND txn_datetime >= now() - INTERVAL 3 MONTH)
    OR (period = '6m' AND txn_datetime >= now() - INTERVAL 6 MONTH)
    OR (period = '1y' AND txn_datetime >= now() - INTERVAL 1 YEAR)
    OR (period = 'all')
GROUP BY
    cust_code,
    acnt_code,
    oper_code,
    period;
```