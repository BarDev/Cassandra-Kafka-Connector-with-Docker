# DSE Kafka Commands

## Check what Docker Containers are running

```
docker ps | awk '{ print $1, $NF }'
```

## Start Docker Compose with DSE, Zookeeper, Kafka

```
docker-compose up -d 
```

## Sanity Check - DSE Working

```
docker exec -it --user root dse bash
nodetool status
```

## Crate Table and Keyspace

```
cqlsh

CREATE KEYSPACE cycling WITH 
replication = {'class': 'SimpleStrategy', 'replication_factor': 1};


CREATE TABLE cycling.comments (
    id UUID,
    created_at TIMESTAMP, 
    comment TEXT,
    commenter TEXT,
    record_id TIMEUUID,
    PRIMARY KEY (id, created_at))
  WITH CLUSTERING ORDER BY (created_at DESC);
  
SELECT * FROM cycling.comments;
```

**********************************
## Sanity Check - Kafka Working

```
docker exec -it --user root kafka bash
```

## Install Curl, Vim, jq

```
apt update && \
apt install curl vim jq -y
```

## Is Kafka Working (Command)

```
kafka-topics.sh --list  --zookeeper zookeeper:2181
kafka-topics.sh --create   --zookeeper zookeeper:2181   --replication-factor 1   --partitions 1   --topic CyclingComments
kafka-topics.sh --list  --zookeeper zookeeper:2181
```

***********************************

## Install DataStax Kafka Connector
https://downloads.datastax.com/#akc

```
mkdir -p ~/dse-connector && \
cd ~/dse-connector && \
mkdir downloads

# curl -L https://downloads.datastax.com/kafka/kafka-connect-dse.tar.gz --output kafka-connect-dse.tar.gz

curl -L https://downloads.datastax.com/kafka/kafka-connect-dse-1.3.1.tar.gz --output downloads/kafka-connect-dse.tar.gz
tar -xzf downloads/kafka-connect-dse.tar.gz    
```

***********************************
## Configure Kafka Connector 

Tell it where the DSE Connector is located connect-distributed.properties

```
vim /opt/bitnami/kafka/config/connect-distributed.properties

cp /opt/bitnami/kafka/config/connect-distributed.properties /opt/bitnami/kafka/config/connect-distributed.properties.bk

# /dse-connector/kafka-connect-dse-1.3.1/kafka-connect-dse-1.3.1.jar

sed -E \
    -e "s|key\.converter\=.*|key.converter=org.apache.kafka.connect.storage.StringConverter|"  \
    -e "s|value\.converter\=.*|value.converter=org.apache.kafka.connect.storage.StringConverter|" \
    -e "s|key\.converter\.schemas\.enable\=.*|key.converter.schemas.enable=false|" \
    -e "s|value\.converter\.schemas\.enable\=.*|value.converter.schemas.enable=false|" \
    -e "s|#plugin.path=|plugin.path=/dse-connector/kafka-connect-dse-1.3.1/kafka-connect-dse-1.3.1.jar|" \
/opt/bitnami/kafka/config/connect-distributed.properties.bk > /opt/bitnami/kafka/config/connect-distributed.properties
```

*******************
## Start Kafka Connector (Command)

```
connect-distributed.sh /opt/bitnami/kafka/config/connect-distributed.properties
```


*******************
## Create DSE Kafka Connector *** Runs in Forground

Login To Kafka Server

```
docker exec -it --user root kafka bash
```


Create DSE Kafka Connector (REST)

```
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{
        "name": "cycling-comments-sink",
        "config": {
          "contactPoints": "dse",
          "loadBalancing.localDc": "dc1",
          "connector.class": "com.datastax.kafkaconnector.DseSinkConnector",
          "tasks.max": "1",
          "topics": "CyclingComments",
          "topic.CyclingComments.cycling.comments.mapping":"record_id=value.rid,id=value.id,commenter=value.author,comment=value.comment,created_at=value.created_at"
        }
      }' \
    http://kafka:8083/connectors
```

*********************

## Valide DSE Kafka Connector Sink was Created (REST)

```
# New Terminal


curl -s GET http://kafka:8083/connectors/ | jq .

# Other Commands
curl -s GET http://kafka:8083/connectors/cycling-comments-sink | jq .
curl -s GET http://kafka:8083/connectors/cycling-comments-sink/config | jq .
curl -s GET http://kafka:8083/connectors/cycling-comments-sink/status | jq .

# curl -X DELETE http://kafka:8083/connectors/cycling-comments-sink | jq .
# curl -s POST http://kafka:8083/connectors/cycling-comments-sink/restart | jq .

```

## Validate Nothing is in DataStax Cycling.Comments table

```
SELECT * FROM cycling.comments;
```

*****************
# Use Kafka Producer to Stream data to Kafka CyclingComments Topic (Command)

```
kafka-console-producer.sh \
--broker-list localhost:9092 \
--topic CyclingComments << EOF
  {"id": "e7ae5cf3-d358-4d99-b900-85902fda9bb0", "created_at": "2017-04-01 14:33:02.160Z", "comment": "LATE RIDERS SHOULD NOT DELAY THE START", "author": "Alex", "rid": "22d496d1-cf24-11e8-a84b-2b44b2d77e7c"}
  {"id": "e7ae5cf3-d358-4d99-b900-85902fda9bb0", "created_at": "2017-03-21 21:11:09.999Z", "comment": "Second rest stop was out of water", "author": "Alex", "rid": "22d38561-cf24-11e8-a84b-2b44b2d77e7c"}
  {"id": "e7ae5cf3-d358-4d99-b900-85902fda9bb0", "created_at": "2017-02-14 20:43:20.234Z", "comment": "Raining too hard should have postponed", "author": "Alex", "rid": "22d225d1-cf24-11e8-a84b-2b44b2d77e7c"}
EOF
```

## Validate Nothing is in DataStax Cycling.Comments table

```
SELECT * FROM cycling.comments;
```


## Validate data is in Kafka (Command)

```
kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic CyclingComments --from-beginning
# <ctrl+c>
```


***********************************
***********************************
***********************************

# OLD Code
## Configure Kafka Connector and tell it where the DSE Connector is located

connect-distributed.properties

```
vim /opt/bitnami/kafka/config/connect-distributed.properties

find / -type f -name connect-distributed.properties 2>/dev/nul

cp /opt/bitnami/kafka/config/connect-distributed.properties /opt/bitnami/kafka/config/connect-distributed.properties.bk

topLevelDir=$(tar -tzf downloads/kafka-connect-dse.tar.gz | head -1 | cut -f1 -d"/")

dseConnectorPath=$(realpath $topLevelDir/$topLevelDir.jar)


sed -E \
    -e "s|key\.converter\=.*|key.converter=org.apache.kafka.connect.storage.StringConverter|"  \
    -e "s|value\.converter\=.*|value.converter=org.apache.kafka.connect.storage.StringConverter|" \
    -e "s|key\.converter\.schemas\.enable\=.*|key.converter.schemas.enable=false|" \
    -e "s|value\.converter\.schemas\.enable\=.*|value.converter.schemas.enable=false|" \
    -e "s|#plugin.path=|plugin.path=$dseConnectorPath|" \
/opt/bitnami/kafka/config/connect-distributed.properties.bk > /opt/bitnami/kafka/config/connect-distributed.properties

connect-distributed.sh /opt/bitnami/kafka/config/connect-distributed.properties
```


