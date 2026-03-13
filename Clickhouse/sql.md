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