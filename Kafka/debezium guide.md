### table 
```sql 
CREATE TABLE C##DBZUSER.DEBEZIUM_SIGNAL  
(  
    id   VARCHAR(42) PRIMARY KEY,  
    type VARCHAR(32)   NOT NULL,  
    data VARCHAR(2048) NULL  
);
```
### insert sql 
```sql 
INSERT INTO  
    C##DBZUSER.DEBEZIUM_SIGNAL (id, type, data)  
VALUES  
    (SYS_GUID(),  
     'execute-snapshot',  
     '{"data-collections": ["MOST_PDB1.NESHUR.DC_HUR_CTZN_SAL"], "type": "INCREMENTAL"}');
```
![[debezium-architecture.png]]
```yaml
{
  "connector.class": "io.debezium.connector.oracle.OracleConnector",
  "incremental.snapshot.chunk.size": "8192",
  "tasks.max": "1",
  "schema.include.list": "C##DBZUSER,NESHUR,NESTDBBANK,NES_PROD",
  "log.mining.transaction.retention.ms": "1800000",
  "internal.log.mining.log.query.max.retries": "15",
  "signal.enabled.channels": "source",
  "schema.history.internal.store.only.captured.tables.ddl": "true",
  "schema.history.internal.store.only.captured.databases.ddl": "true",
  "topic.prefix": "most_pdb1",
  "decimal.handling.mode": "string",
  "schema.history.internal.kafka.topic": "schema-changes.most_pdb1",
  "signal.data.collection": "MOST_PDB1.C##DBZUSER.DEBEZIUM_SIGNAL",
  "log.mining.archive.log.only.mode": "false",
  "database.user": "c##dbzuser",
  "database.dbname": "most",
  "signal.kafka.bootstrap.servers": "kafka:9092",
  "database.pdb.name": "most_pdb1",
  "incremental.snapshot.watermarking.strategy": "insert_delete",
  "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
  "event.processing.failure.handling.mode": "warn",
  "database.port": "1521",
  "internal.log.mining.log.backoff.initial.delay.ms": "3000",
  "database.hostname": "192.168.123.65",
  "database.password": "dbz",
  "internal.log.mining.log.backoff.max.delay.ms": "120000",
  "name": "most_pdb1",
  "table.include.list": "NESHUR.DC_HUR_CTZN_SAL,C##DBZUSER.DEBEZIUM_SIGNAL,NES_PROD.MG_CUST_ACNT,NES_PROD.MG_CUST_ATTR",
  "snapshot.mode": "initial"
}
```
