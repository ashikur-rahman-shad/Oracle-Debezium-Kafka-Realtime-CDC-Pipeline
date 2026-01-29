Remember, if you get permission errors, use `sudo` in shell

## Download Kafka from website and Extract it (on all servers)
```bash
cd Downloads/
wget https://downloads.apache.org/kafka/4.1.1/kafka_2.13-4.1.1.tgz 
sudo tar -xzf kafka_2.13-4.1.1.tgz -C /opt 
sudo mv /opt/kafka_2.13-4.1.1 /opt/kafka
cd /opt/kafka
```
You can install it on any folder literally. opt/kafka is recommended

## Configure server properties for each server

Edit `config/server.properties`:
```yaml
node.id= set your unique number
controller.quorum.voters=<node id of server1>@<ip of server1>:9093,<node id of server2>@<ip of server2>:9093,<node id of server3>@<ip of server3>:9093
listeners=PLAINTEXT://<CURRENT SERVER IP>:9092,CONTROLLER://<CURRENT SERVER IP>:9093
advertised.listeners=PLAINTEXT://<CURRENT SERVER IP>:9092,CONTROLLER://<CURRENT SERVER IP>:9093
log.dirs= #set log directory to anywhere other than temp folder

## Comment this line
#controller.quorum.bootstrap.servers=<IP_1>:9093,<IP_2>:9093,<IP_3>:9093

## The followings may already be fine
process.roles=broker,controller
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
controller.listener.names=CONTROLLER 
```

---

## Configure KRaft

### Generate key (DO THIS ONCE ONLY, and use the same cluster ID on all servers)

For bash/zsh
```bash
KAFKA_CLUSTER_ID=$(./bin/kafka-storage.sh random-uuid)
```
For fish shell
```fish
set KAFKA_CLUSTER_ID $(./bin/kafka-storage.sh random-uuid)
```
Check the Cluster ID (Optional)
```bash
echo "$KAFKA_CLUSTER_ID"
```

***==Save this cluster ID==***

---

### Format Storage (Do this on all servers)

Stop server if running
```bash
sudo ./bin/kafka-server-stop.sh
```
Delete older logs if exists
```bash
sudo rm -rf ./kraft-combined-logs/
sudo rm -rf ./kafka-logs
```

Format the log directory (multi-nodes). Make sure all clusters use the SAME CLUSTER ID
```bash
sudo ./bin/kafka-storage.sh format -t KAFKA_CLUSTER_ID -c ./config/server.properties
```
Example:
sudo ./bin/kafka-storage.sh format -t yADtBHEZQG6Y13LVUdQCtA -c ./config/server.properties

### Allow firewall (Do this on all servers)
```
sudo firewall-cmd --add-port=9092/tcp --permanent
sudo firewall-cmd --add-port=9093/tcp --permanent
sudo firewall-cmd --reload
```

## Start Kafka Server (Do this on all servers)
```bash
sudo bin/kafka-server-start.sh config/server.properties
```

To Stop Server:
```bash
# if running in background
sudo bin/kafka-server-stop.sh config/server.properties

# to force stop, find the process, 
sudo lsof -i :9093
# then force kill it
sudo kill -9 <PID>
```

View cluster ID in future:
```bash
/opt/kafka/bin/kafka-cluster.sh cluster-id --bootstrap-server localhost:9092
```
*You can also use broker address instead of  localhost*

## Create Kafka Service (Do this on all servers)
This will autostart kafka with boot.

Edit this file
```bash
sudo nano /etc/systemd/system/kafka.service
```
sudo micro /etc/systemd/system/kafka.service

Make sure to update the ExecStart and ExecStop based on your kafka installation location
```ini
[Unit]
Description=Apache Kafka Server
Documentation=http://kafka.apache.org/documentation.html
Requires=network.target

[Service]
Type=simple
User=root
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.8.0.9-1.0.1.el8.x86_64/"
ExecStart=/u02/kafka/kafka_2.13-4.1.1/bin/kafka-server-start.sh /u02/kafka/kafka_2.13-4.1.1/config/server.properties
ExecStop=/u02/kafka/kafka_2.13-4.1.1/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Check if SELinux is enabled or not:
```bash
sestatus
```

If enabled, allow your kafka installation locations (this may require when running binary from unusual locations)
```bash
# 1. Label the binaries as executable service files
sudo semanage fcontext -a -t bin_t "/u02/kafka/kafka_2.13-4.1.1/bin(/.*)?"

# 2. Label the logs and data directory (if separate)
sudo semanage fcontext -a -t var_log_t "/u02/kafka/kafka_2.13-4.1.1/kraft-logs(/.*)?"

# 3. Apply these changes to the actual files
sudo restorecon -Rv /u02/kafka/
```

Activate service
```bash
sudo systemctl daemon-reload
sudo systemctl start kafka
sudo systemctl enable kafka
```

# Kafka UI (can be installed on one or more servers)

## Pre-requisites

### Install docker
```bash
sudo apt install -y docker
```

Docker version we used
```bash
docker --version
Docker version 28.5.1, build e180ab8
```

To run docker without sudo:
```bash
sudo usermod -aG docker (whoami)
```


## Installing Kafka-UI (using docker)

Create docker-compose.yaml file
```bash
mkdir ~/kafka-ui; cd ~/kafka-ui
editor docker-compose.yaml
```

Add your configuration to the docker-compose.yaml file. Fill up `YOUR_IP`!
```yaml
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    network_mode: "host"
    environment:
      KAFKA_CLUSTERS_0_NAME: local-dgx
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: 10.11.200.99:9092
      DYNAMIC_CONFIG_ENABLED: 'true'
```

Install Kafka-UI
```bash
sudo docker compose up -d
```


# Testing

Check if cluster is reachable or not (optional)
```bash
/opt/kafka/bin/kafka-broker-api-versions.sh \
--bootstrap-server 10.11.200.99:9092
```

**Doing the following on one server will sync changes to all servers in the cluster**

### Create Topic
```bash
/opt/kafka/bin/kafka-topics.sh \
--create \
--topic test-topic \
--bootstrap-server 10.11.200.99:9092 \
--partitions 3 \
--replication-factor 3
```
Here, 
- you can use any bootstrap-server address from your cluster's broker list
- Partition allows to deal with consumers parallel increasing performance, but takes more memory and space
- Replication will allow you to replicate the data across multiple brokers. Even if the leader server fail, next server will become the leader. Max limit: Number of brokers.

###  Create Consumer
```bash
/opt/kafka/bin/kafka-console-consumer.sh \
--topic test-topic \
--from-beginning \
--bootstrap-server 10.11.205.205:9092
```

### Create Producer
```bash
/opt/kafka/bin/kafka-console-producer.sh \
--topic test-topic \
--bootstrap-server 10.11.200.99:9092
```

Test Multiple Consumer
```bash
/opt/kafka/bin/kafka-console-consumer.sh \
--topic test-topic \
--group test-group \
--bootstrap-server 10.11.200.99:9092 \
--property print.partition=true \
--property print.offset=true
```
