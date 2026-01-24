### Most Bwl
```json 
{
  "connector.class": "io.debezium.connector.oracle.OracleConnector",
  "snapshot.include.collection.list": "MOLE_PDB.NES_FMS.BWL_LIST",
  "database.user": "c##dbzuser",
  "database.dbname": "MOLEORCL",
  "tasks.max": "1",
  "database.pdb.name": "MOLE_PDB",
  "schema.include.list": "NES_FMS",
  "log.mining.transaction.retention.ms": "1800000",
  "internal.log.mining.log.query.max.retries": "15",
  "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
  "database.port": "1521",
  "schema.history.internal.store.only.captured.tables.ddl": "true",
  "schema.history.internal.store.only.captured.databases.ddl": "true",
  "topic.prefix": "BWL_LIST",
  "internal.log.mining.log.backoff.initial.delay.ms": "3000",
  "schema.history.internal.kafka.topic": "schema-changes.BWL_LIST",
  "database.hostname": "192.168.127.35",
  "database.schema": "NES_FMS",
  "internal.log.mining.log.backoff.max.delay.ms": "120000",
  "database.password": "y2#6#8Md#kp0j4T",
  "name": "BWL_LIST",
  "table.include.list": "NES_FMS.BWL_LIST",
  "database.oracle.version": "19",
  "snapshot.mode": "initial"
}
```

### Most pdb1
``` json
{
  "connector.class": "io.debezium.connector.oracle.OracleConnector",
  "database.user": "c##dbzuser",
  "database.dbname": "most",
  "tasks.max": "1",
  "database.pdb.name": "most_pdb1",
  "schema.include.list": "NES_PROD",
  "log.mining.transaction.retention.ms": "1800000",
  "internal.log.mining.log.query.max.retries": "15",
  "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
  "database.port": "1521",
  "schema.history.internal.store.only.captured.tables.ddl": "true",
  "schema.history.internal.store.only.captured.databases.ddl": "true",
  "topic.prefix": "most_pdb1",
  "decimal.handling.mode": "string",
  "internal.log.mining.log.backoff.initial.delay.ms": "3000",
  "schema.history.internal.kafka.topic": "schema-changes.most_pdb1",
  "database.hostname": "192.168.123.65",
  "database.password": "dbz",
  "internal.log.mining.log.backoff.max.delay.ms": "120000",
  "name": "most_pdb1",
  "table.include.list": "NES_PROD.MG_CUST_ACNT,NES_PROD.MG_CUST_ACNT_LINK",
  "log.mining.archive.log.only.mode": "false",
  "snapshot.mode": "initial"
}
```