```sql 
-- Transactions table equivalent  
CREATE TABLE cad_transactions (  
    cust_code  VARCHAR2(20)    NOT NULL,  
    acnt_code  VARCHAR2(20)    NOT NULL,  
    txn_time   TIMESTAMP       NOT NULL,  
    txn_amt    NUMBER(18, 2)   NOT NULL,  
    is_credit  NUMBER(1)       NOT NULL CHECK (is_credit IN (0,1)),  
    oper_code  VARCHAR2(20)    NOT NULL  
)  
PARTITION BY RANGE (txn_time)  
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))  
(PARTITION p_initial VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00'))  
COMPRESS FOR OLTP;  
  
-- Indexes (mirrors ClickHouse ORDER BY key)  
CREATE INDEX idx_txn_cust_acnt_time  
    ON cad_transactions (cust_code, acnt_code, txn_time)  
    LOCAL;  -- partition-local index  
  
-- Aggregation table  
CREATE TABLE cad_aggregated_acnt_metrics (  
    cust_code          VARCHAR2(20),  
    acnt_code          VARCHAR2(20),  
    period             VARCHAR2(2)  CHECK (period IN ('1m','3m','6m','1y','all')),  
    oper_code          VARCHAR2(20),  
    txn_count          NUMBER,  
    txn_amt_sum        NUMBER(18,2),  
    credit_txn_amt_sum NUMBER(18,2),  
    debit_txn_amt_sum  NUMBER(18,2),  
    txn_amt_avg        NUMBER(18,4),  
    txn_amt_min        NUMBER(18,2),  
    txn_amt_max        NUMBER(18,2),  
    last_updated       TIMESTAMP DEFAULT SYSTIMESTAMP,  
    CONSTRAINT pk_acnt_metrics PRIMARY KEY (cust_code, acnt_code, oper_code, period)  
);
```


```sql 
-- Fast-refresh MV (closest Oracle equivalent to ClickHouse MV)  
-- Requires a materialized view log on the base table  
CREATE MATERIALIZED VIEW LOG ON cad_transactions  
    WITH ROWID, SEQUENCE  
    (cust_code, acnt_code, txn_time, txn_amt, is_credit, oper_code)  
    INCLUDING NEW VALUES;  
  
CREATE MATERIALIZED VIEW cad_mv_acnt_metrics  
BUILD DEFERRED  
REFRESH COMPLETE ON DEMAND  
-- no ENABLE QUERY REWRITE  
AS  
WITH periods AS (  
    SELECT  
        cust_code, acnt_code, txn_time, txn_amt, is_credit, oper_code,  
        COLUMN_VALUE AS period  
    FROM cad_transactions  
    CROSS JOIN TABLE(SYS.ODCIVARCHAR2LIST('1m','3m','6m','1y','all'))  
)  
SELECT  
    cust_code,  
    acnt_code,  
    period,  
    oper_code,  
    COUNT(*)                                         AS txn_count,  
    SUM(txn_amt)                                     AS txn_amt_sum,  
    SUM(CASE WHEN is_credit = 1 THEN txn_amt END)    AS credit_txn_amt_sum,  
    SUM(CASE WHEN is_credit = 0 THEN txn_amt END)    AS debit_txn_amt_sum,  
    AVG(txn_amt)                                     AS txn_amt_avg,  
    MIN(txn_amt)                                     AS txn_amt_min,  
    MAX(txn_amt)                                     AS txn_amt_max  
FROM periods  
WHERE  
    (period = '1m'  AND txn_time >= TRUNC(SYSDATE, 'MM'))  
    OR (period = '3m'  AND txn_time >= ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -2))  
    OR (period = '6m'  AND txn_time >= ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -5))  
    OR (period = '1y'  AND txn_time >= TRUNC(SYSDATE, 'YEAR'))  
    OR (period = 'all')  
GROUP BY cust_code, acnt_code, oper_code, period;
```

```sql 
CREATE MATERIALIZED VIEW LOG ON cad_transactions  
    WITH ROWID, SEQUENCE  
    (cust_code, acnt_code, txn_time, txn_amt, is_credit, oper_code)  
    INCLUDING NEW VALUES;
```