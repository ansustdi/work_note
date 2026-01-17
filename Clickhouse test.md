```sql
SELECT *  
FROM  
    wh.casa_fintxn AS ct  
        INNER JOIN wh.casa_acnt AS ca ON ca.acnt_code = ct.acnt_code  
        LEFT JOIN wh.tmw_jrnl AS t ON t.jrno = ct.jrno  
WHERE  
    ct.txn_date = '2025-12-22'  
LIMIT 5;
```


```sql
WITH  
    ct AS (SELECT *  
    FROM  
        wh.casa_fintxn  
    WHERE  
        txn_date = '2025-12-22')  
SELECT ca.cust_code, sum(ct.txn_amount)  
FROM  
    ct  
        INNER JOIN wh.casa_acnt ca ON ca.acnt_code = ct.acnt_code  
GROUP BY  
    ca.cust_code;
```

```sql
WITH  
    ct AS (SELECT *  
    FROM  
        wh.casa_fintxn  
    WHERE  
        txn_date = '2025-12-22'),  
    t AS (SELECT *  
    FROM  
        wh.tmw_jrnl  
    WHERE  
        txn_date = '2025-12-22')  
SELECT ca.cust_code, sum(ct.txn_amount)  
FROM  
    ct  
        INNER JOIN wh.casa_acnt ca ON ca.acnt_code = ct.acnt_code  
        LEFT JOIN t ON t.jrno = ct.jrno  
GROUP BY  
    ca.cust_code;
```  
  