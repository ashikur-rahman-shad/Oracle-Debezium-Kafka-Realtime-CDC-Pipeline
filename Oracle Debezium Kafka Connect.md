## Create Kafka Connect Internal Topics (Doing it on one server address will sync all servers in the cluster)

Offset Topic
```bash
bin/kafka-topics.sh --bootstrap-server 10.11.200.99:9092 \
  --create --topic connect-offsets \
  --partitions 50 \
  --replication-factor 3 \
  --config cleanup.policy=compact
```

Config Topic
```bash
bin/kafka-topics.sh --bootstrap-server 10.11.200.99:9092 \
  --create --topic connect-configs \
  --partitions 1 \
  --replication-factor 3 \
  --config cleanup.policy=compact
```

Status Topic
```bash
bin/kafka-topics.sh --bootstrap-server 10.11.200.99:9092 \
  --create --topic connect-status \
  --partitions 10 \
  --replication-factor 3
  --config cleanup.policy=compact
```
Never use Replication Factor less than 3 for industrial use

---

## Download & Install Debezium Oracle Plugin (Do this for all servers)
```bash
mkdir -p plugins && cd plugins
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-oracle/3.4.0.Final/debezium-connector-oracle-3.4.0.Final-plugin.tar.gz
tar -xzf debezium-connector-oracle-3.4.0.Final-plugin.tar.gz
## Remove the tarball to free up space (Optional)
rm debezium-connector-oracle-3.4.0.Final-plugin.tar.gz
```

## Kafka Connect (Do these on all servers)

Edit connect-distributed.properties
```properties
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
group.id=connect-oracle-cdc

offset.storage.topic=connect-offsets
config.storage.topic=connect-configs
status.storage.topic=connect-status

offset.flush.interval.ms=10000

plugin.path=/opt/kafka/plugins

rest.port=8083
rest.advertised.host.name=<this-node-hostname>
```

EOS is enforced at the Kafka Connect worker level. Add these lines to connect-distributed.properties:
```properties
producer.override.acks=all
producer.override.enable.idempotence=true
producer.override.max.in.flight.requests.per.connection=5
```

- This guarantees:
- No duplicate events
- No lost commits
- Safe retries

---

## Start Kafka Connect Cluster (On all servers)

Start on all workers:
```bash
./bin/connect-distributed.sh ./config/connect-distributed.properties
```

Verify:
```bash
curl http://connect1:8083/connectors
```

### Register Kafka Connectors (Do it once only)
```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
  http://localhost:8083/connectors/ -d '{
  "name": "oracle-de-transactions-connector",
  "config": {
    "connector.class": "io.debezium.connector.oracle.OracleConnector",
    "tasks.max": "1",
    "database.user": "C##DEBEZIUM",
    "database.password": "Debezium2026#",
    "database.url": "jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=10.11.200.37)(PORT=1521))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=ORCLCDB)))",
    "database.dbname": "ORCLCDB",
    "database.pdb.name": "ORCLPDB1",
    "topic.prefix": "cdc_oracle",
    "table.include.list": "DE.TRANSACTIONS",
    "decimal.handling.mode": "double",
    "schema.history.internal.kafka.bootstrap.servers": "10.11.200.99:9092,10.11.200.37:9092,10.11.205.206:9092",
    "schema.history.internal.kafka.topic": "schema-changes.de",
    "database.connection.adapter": "logminer",
    "snapshot.mode": "initial"
  }
}'
```

### Checking status
```bash
curl -X POST http://connect1:8083/connectors \
  -H "Content-Type: application/json" \
  -d @oracle19c-cdc.json
```

```bash
curl -s http://10.11.200.99:8083/connectors/connect-oracle-cdc/status | jq
```
