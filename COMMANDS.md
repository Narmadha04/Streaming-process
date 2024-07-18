#### To compose a docker file:
```sh
docker-compose up –d
```
#### To alter a table in postgres to modify the identity to full:
replace table_name with your table name
```sh
ALTER TABLE public.table_name REPLICA IDENTITY FULL;
```
#### To create a debezium connector:
```sh
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 127.0.0.1:8083/connectors/ --data "@debezium.json"
```
#### To create partitions
replace 3 with number of partitions you need
```sh
docker exec -it kafka1-kafka-1 /bin/bash
kafka-topics --bootstrap-server localhost:9092 --alter --topic postgres.public.table_name --partitions 3
```
#### To list the networks available
```sh
docker network ps
```
#### To run a kafkacat when debezium uses avro convertor
replace network_name and table_name with your network name and table name
```sh
docker run --tty --network network_name confluentinc/cp-kafkacat kafkacat -b kafka:9092 -C -s value=avro -r http://schema-registry:8081 -t postgres.public.table_name
```
#### To run a kafkacat when debezium uses json convertor
replace network_name and table_name with your network name and table name
```sh
docker run --tty --network network_name confluentinc/cp-kafkacat kafkacat -b kafka:9092 -C -f '%s\n' -t postgres.public.table_name
```
#### To list the topics
```sh
docker-compose exec kafka kafka-topics --list --bootstrap-server kafka:9092
```
#### To list the topics's meta data
replace network_name with your network name
```sh
docker run --network=network_name --rm confluentinc/cp-kafkacat kafkacat -b kafka:9092 –L
```
#### To delete a topic
replace network_name and table_name with your network name and table name
```sh
docker exec -it kafka1-kafka-1 /bin/bash
kafka-topics --delete --bootstrap-server localhost:9092 --topic postgres.public.table_name
```
#### To list the containers
```sh
docker ps
```
#### To get into a particular container's bash
replace connector_id with your connector id
```sh
docker exec -it container_id bash
```
#### To list all the containers
```sh
curl -X GET http://localhost:8083/connectors
```
#### To get the status of a connector
replace connector_name with your connector_name
```sh
curl -X GET http://localhost:8083/connectors/connector_name/status
```
#### To delete a connector
replace connector_name with your connector_name
```sh
curl -X DELETE http://localhost:8083/connectors/connector_name
```
#### To list the schemas that are in schema registry
```sh
curl -s 127.0.0.1:8081/subjects
```

