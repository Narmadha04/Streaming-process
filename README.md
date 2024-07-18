# Analysis with Kafka, Debezium, and PySpark

This project demonstrates how to use Kafka, Debezium, and PySpark for real-time change data capture (CDC) from a PostgreSQL database. The setup includes Docker services for Kafka, Zookeeper, Debezium, Schema Registry, and a PySpark Jupyter notebook.

## Prerequisites

- Docker 
- Basic knowledge of Kafka, PostgreSQL, and PySpark
- [https://medium.com/@narmadhabts/apache-kafka-25028fb95bfd](url)

## Project Structure

- `docker-compose.yaml`: Docker Compose file to set up the required services
- `debezium.json`: Debezium connector configuration

## Setup

You can download the docker-compose.yaml and debezium.json files manually or follow step 1

### Step 1: Clone the Repository
Run the following commands to clone the repository and navigate to the project directory:

```sh
git clone https://github.com/yourusername/directory_name.git
cd directory_name
```
### Step 2: Start the Docker Services
>[!NOTE]
>- Open a command prompt or terminal.
>- Navigate to your project directory.
  
Run the following command to start the Docker services defined in docker-compose.yaml:

```sh
docker-compose up -d
```
### Step 3: Configure PostgreSQL Table
Ensure that the table that you are trying to connect in your PostgreSQL database has the replica identity set to FULL replace with your table name:

```sql
ALTER TABLE public.table_name REPLICA IDENTITY FULL;
```
### Step 4: Set Up the Debezium Connector
>[!IMPORTANT]
>Update the connection parameters in debezium.json as needed:
>
>- database.hostname
>- database.port
>- database.user
>- database.password
>- database.dbname
>- table.include.list
  
Then, register the connector with Debezium:

```sh
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://localhost:8083/connectors/ --data "@debezium.json"
```
### Step 5: Verify Kafka Topics
List the networks in Docker to see the name of the network:

```sh
docker network ls
```
Consume messages from Kafka to verify that the connector is working. 
>[!TIP]
>Kafka cat, also known as kcat, is a command-line utility that allows users to interact with Apache Kafka topics.

Replace with your network name which you can find from the output of the previous command:

```sh
docker run --tty --network network_name confluentinc/cp-kafkacat kafkacat -b kafka:9092 -C -f '%s\n' -t postgres.public.table_name
```
This kafkacat environment should be running to see the changes made in database.
### Step 6: Insert Sample Data
Insert sample data into the public.table_name to generate change events
  
### Step 7: Run PySpark Consumer
Open the Jupyter notebook in your browser (usually at http://localhost:8888), you can find the token in your pyspark logs and create a password in your localhost 8888, and run the PySpark code provided to start consuming Kafka messages:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, LongType
from IPython.display import clear_output
import threading
import time

#spark session builder
spark = SparkSession.builder \
    .appName("KafkaSparkConsumer") \
    .getOrCreate()

#schema of the table
after_schema = StructType([
    StructField("patientid", IntegerType(), False),
    StructField("firstname", StringType(), True),
    StructField("lastname", StringType(), True),
    StructField("dob", IntegerType(), True),
    StructField("gender", StringType(), True),
    StructField("contactnumber", StringType(), True),
    StructField("email", StringType(), True),
    StructField("address", StringType(), True),
    StructField("city", StringType(), True)
])

#schema of the kafka 
schema = StructType([
    StructField("before", StringType(), True),
    StructField("after", after_schema, True),
    StructField("source", StringType(), True),
    StructField("op", StringType(), True),
    StructField("ts_ms", LongType(), True),
    StructField("transaction", StringType(), True)
])

df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "postgres.public.table_name") \
    .option("startingOffsets", "earliest") \
    .load()

df = df.selectExpr("CAST(value AS STRING) as json_string")

df = df.select(from_json(col("json_string"), schema).alias("data")).select("data.after.*")

memory_query = df.writeStream \
    .outputMode("append") \
    .format("memory") \
    .queryName("patients_table") \
    .start()

def display_output(freq=5):
    while True:
        clear_output(wait=True)
        patients_df = spark.sql("SELECT * FROM patients_table")
        patients_df.show(n=1000, truncate=False)
        time.sleep(freq)  

display_thread = threading.Thread(target=display_output, args=(5,))
display_thread.start()

memory_query.awaitTermination()
```
This code will read changes from the postgres.public.table_name topic in Kafka, process them with PySpark, and display the data in real-time.


<details>
  <summary>Commands (For reference)</summary>
  
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
</details>

