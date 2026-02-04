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