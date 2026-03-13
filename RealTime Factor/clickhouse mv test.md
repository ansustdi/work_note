
```sql 
CREATE TABLE cad.transactions
(
    `cust_code`    String,
    `acnt_code`    String,
    `txn_time`     DateTime,
    `txn_amt`      Decimal(18, 2),
    `is_credit`    UInt8, -- 1 for credit, 0 for debit
    `oper_code`    String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(txn_time)
ORDER BY (cust_code, acnt_code, txn_time);
```

```sql
CREATE TABLE cad.aggregated_acnt_metrics
(
    `cust_code`              String,
    `acnt_code`              String,
    `period`                 Enum8('1m' = 1, '3m' = 2, '6m' = 3, '1y' = 4, 'all' = 5),
    `oper_code`              String,

    -- Aggregate states
    `txn_count`              AggregateFunction(count),
    `txn_amt_sum`            AggregateFunction(sum, Decimal(18, 2)),
    `credit_txn_amt_sum`     AggregateFunction(sumIf, Decimal(18, 2), UInt8),
    `debit_txn_amt_sum`      AggregateFunction(sumIf, Decimal(18, 2), UInt8),
    `txn_amt_avg`            AggregateFunction(avg, Decimal(18, 2)),
    `txn_amt_min`            AggregateFunction(min, Decimal(18, 2)),
    `txn_amt_max`            AggregateFunction(max, Decimal(18, 2)),

    `last_updated`           DateTime DEFAULT now()
)
ENGINE = AggregatingMergeTree()
ORDER BY (cust_code, acnt_code, oper_code, period);
```

```sql
CREATE MATERIALIZED VIEW cad.mv_acnt_metrics TO cad.aggregated_acnt_metrics  
AS  
WITH  
    periods AS (SELECT  
        cust_code,  
        acnt_code,  
        txn_time,  
        txn_amt,  
        is_credit,  
        oper_code,  
        arrayJoin(['1m', '3m', '6m', '1y', 'all']) AS period  
    FROM  
        cad.transactions)  
SELECT  
    cust_code,  
    acnt_code,  
    period,  
    oper_code,  
  
    -- State functions to create the intermediate aggregation states  
    countState() AS txn_count,  
    sumState(txn_amt) AS txn_amt_sum,  
    sumIfState(txn_amt, is_credit = 1) AS credit_txn_amt_sum,  
    sumIfState(txn_amt, is_credit = 0) AS debit_txn_amt_sum,  
    avgState(txn_amt) AS txn_amt_avg,  
    minState(txn_amt) AS txn_amt_min,  
    maxState(txn_amt) AS txn_amt_max  
FROM  
    periods  
WHERE  
    (period = '1m' AND txn_time >= toStartOfMonth(now()))  
    OR (period = '3m' AND txn_time >= toStartOfMonth(now() - INTERVAL 2 MONTH))  
    OR (period = '6m' AND txn_time >= toStartOfMonth(now() - INTERVAL 5 MONTH))  
    OR (period = '1y' AND txn_time >= toStartOfYear(now()))  
    OR (period = 'all')  
GROUP BY  
    cust_code,  
    acnt_code,  
    oper_code,  
    period;
```

```sql 
SELECT  
    cust_code,  
    acnt_code,  
    period,  
    oper_code,  
    countMerge(txn_count) AS total_transactions,  
    sumMerge(txn_amt_sum) AS total_amount  
FROM  
    cad.aggregated_acnt_metrics  
WHERE  
    cust_code = 'a'  
    AND acnt_code = '1'  
    AND period = '1m'  
GROUP BY  
    cust_code, acnt_code, oper_code, period;
```

```sql 
SELECT  
    cust_code,  
    acnt_code,  
    period,  
    oper_code,  
    sumIfMerge(credit_txn_amt_sum) AS total_credit,  
    sumIfMerge(debit_txn_amt_sum) AS total_debit  
FROM  
    cad.aggregated_acnt_metrics  
WHERE  
    cust_code = 'a'  
    AND acnt_code = '1'  
    AND period = '3m'  
GROUP BY  
    cust_code, acnt_code, oper_code, period;
```

```sql 
INSERT INTO cad.transactions
SELECT
    -- cust_code: ~500k unique customers. Every 5 accounts share a customer.
    'CUST_' || toString(intDiv(number % 2500000, 5)) AS cust_code,

    -- acnt_code: 2.5 million unique accounts, cycling through.
    'ACC_' || toString(number % 2500000) AS acnt_code,

    -- txn_time: A random timestamp within the last year.
    (now() - INTERVAL 1 YEAR) + toIntervalSecond(rand() % (365 * 24 * 60 * 60)) AS txn_time,

    -- txn_amt: A skewed distribution, CORRECTED with toDecimal64.
    toDecimal64(1 + pow(rand() / exp2(64), 4) * 5000, 2) AS txn_amt,

    -- is_credit: A 40% chance of being a credit (1), 60% for debit (0).
    if(rand() % 100 < 40, 1, 0) AS is_credit,

    -- oper_code: Randomly pick from a predefined list of categories.
    ['payment', 'deposit', 'purchase', 'utility', 'transfer', 'refund', 'salary'][1 + (rand() % 7)] AS oper_code
FROM
    system.numbers
LIMIT 100000000;
```

```sql
```