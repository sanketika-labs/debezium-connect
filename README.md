# Debezium Kafka Connect
Helm chart for Debezium Kafka Connect

## Pre-requisites

1. Create a postgres user with replication role assigned
```sql
CREATE USER debezium_user WITH REPLICATION ENCRYPTED PASSWORD 'debezium123';
```
2. Create a logical replication slot
```sql
SELECT * FROM pg_create_logical_replication_slot('debezium_replication_slot', 'pgoutput');
```
3. Validate the replication slot
```sql
select * from pg_replication_slots;
```
4. Create a Publication for the list of tables that needs replication into Kafka
```sql
\c <database_name>;
CREATE PUBLICATION debezium_publication FOR TABLE table1, table2;
```

## Installation
1. git clone the debezium connect repo 
```bash
git clone debezium-connect
cd debezium-connect
```
2. Modify the docker image details in the values.yaml file
```yaml
image:
  registry: docker.io
  repository: debezium/connect
  tag: 3.0.0.Final
```

3. Modify the Kafka Bootstrap Servers, Heap Options (memory) and Log level in the values.yaml file
```yaml
env:
  GROUP_ID: 01544
  CONFIG_STORAGE_TOPIC: debezium-connect-config
  OFFSET_STORAGE_TOPIC: debezium-connect-offset
  STATUS_STORAGE_TOPIC: debezium-connect-status
  BOOTSTRAP_SERVERS: 192.168.106.1:9092
  OFFSET_FLUSH_INTERVAL_MS: 60000
  HEAP_OPTS: "-Xms256M -Xmx1G"
  LOG_LEVEL: INFO
```

4. Modify the pod requests and limits in the resources section
```yaml
resources:
  limits:
    cpu: 1
    memory: 1024Mi
  requests:
    cpu: 500m
    memory: 512Mi
```

5. Use Helm to install the Debezium Kafka Connect
```bash
helm install debezium-connect . -n debezium --create-namespace
```

## Configure the Debezium Connector for replicating data to Kafka

Change the following parameters in the `curl` request

1. name (Debezium connector name)
2. database.hostname
3. database.port
4. database.user
5. database.password
6. database.dbname
7. topic.prefix
8. slot.name (Use the replication slot created above)
9. publication.name (Use the publication created above)
10. table.include.list (List of tables to be included in the replication)

The following curl request will register the connector with the name `debezium-connector`
```bash
curl --request POST \
  --url http://<debezium_connect_ip>:8083/connectors \
  --header 'Content-Type: application/json' \
  --data '{
  "name": "debezium-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "192.168.106.1",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "debezium123",
    "database.dbname": "<database_name>",
    "database.server.name": "debezium-server",
    "topic.prefix": "debezium",
    "slot.name": "debezium_replication_slot",
    "publication.name": "debezium_publication",
    "plugin.name": "pgoutput",
    "table.include.list": "public.table1, public.table2",
    "tasks.max": "1",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
  }
}'
```

2. Use the following api to check the status of the connector. It will produce the output shown below
```bash
# Status API
curl -XGET http://localhost:8083/connectors/<connector-name>/status
# Output
{
  "connector": {
    "state": "RUNNING",
    "worker_id": "10.42.0.24:8083"
  },
  "name": "debezium-connector",
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "10.42.0.24:8083"
    }
  ],
  "type": "source"
}
```

## Verify replication in Kafka

1. All INSERTS, UPDATES and DELETES in the tables included in the connector will have the replication created in a separate topic for each table.
2. For e.g., for public.Table1 and public.Table2, the following Kafka topics will be created
```bash
debezium.public.table1
debezium.public.table2
```




