# load-csv2kafka
Load a csv file to kafka via source strema file connector

This is a short example how to load csv files into kafka via File Connector. Later I will transform the csv format (delimiter) into Avro format.
I also register a schema. I use this schema later if I will get avro data and load direct avro data into kafka

## start confluent local and create topics

```
confluent local start
confluent local list connectors

kafka-topics --create --topic car-data-csv --partitions 1 --replication-factor 1 --zookeeper 127.0.0.1:2181
kafka-topics --create --topic cardata --partitions 1 --replication-factor 1 --zookeeper 127.0.0.1:2181
```
## Check schema registry and register schema
The Schema file is load into Schema Registry via python script. I found a nice utitiy here: [Kafka Tutorial](https://aseigneurin.github.io/2018/08/02/kafka-tutorial-4-avro-and-schema-registry.html). It is a python script to register schema and combine it with my topic cardata.
```
# check subjects in schema registry for schema registry
curl http://localhost:8081/subjects/
# register schema  <schema registry REST> <topic> <avro schema file>
python3 register_schema.py http://localhost:8081 cardata cardata-v1.avsc
# check
curl http://localhost:8081/subjects/cardata-value/versions/
# get schema
curl http://localhost:8081/subjects/cardata-value/versions/1
```

Another simple way to register a Schema to a topic is using the Confluent Control Center. Create your topic and add your schema syntax into Schema editor. This Schema is than registered into Schema Registry.

## Load csv into kafka
I use a the file stream source connector as standalone. Configuration files are attached.
```
# check current properties
cat file_stream_demo_standalone.properties
cat worker.properties

# start the standa alone connector
connect-standalone worker.properties file_stream_demo_standalone.properties
# stop with control+c
```

Now you can check if data is in Kafka topic:
```
# check data
kafka-console-consumer --topic users --from-beginning --bootstrap-server 127.0.0.1:9092
kafkacat -b localhost:9092 -t car-data-csv -C
```
## Transform csv to AVRO with KSQL
To transform csv (STRING) Format into AVRO is very easy with KSQL:
```
# transform to AVRO
KSQL> SET 'auto.offset.reset'='earliest';
KSQL> PRINT 'car-data-csv' FROM BEGINNING;
KSQL> create stream car_stream (created varchar, carid varchar, coolant_temp double, intake_air_temp double, intake_air_flow_speed double, battery_percentage double, battery_voltage double, current_draw double, Speed double, engine_vibration_amplitude double, throttle_pos double, tire_pressure_1_1 int, tire_pressure_1_2 int, tire_pressure_2_1 int, tire_pressure_2_2 int, accelerometer_1_1_value double, accelerometer_1_2_value double, accelerometer_2_1_value double,  accelerometer_2_2_value double, control_unit_firmware int) WITH (KAFKA_TOPIC='car-data-csv', VALUE_FORMAT='DELIMITED'); 
KSQL> describe car_stream;
KSQL> SET 'auto.offset.reset' = 'earliest';
KSQL> select * from car_stream;
KSQL> CREATE STREAM car_stream_avro WITH (VALUE_FORMAT='AVRO') AS SELECT * FROM car_stream;
KSQL> describe car_stream_avro;
KSQL> select * from car_stream_avro;
```
The Schema is automatically created in Schema Registry, to check follow the REST Calls:
```
# check automatically created schema
curl http://localhost:8081/subjects/kcmi-CAR_STREAM_AVRO-value/versions/
# get schema
curl http://localhost:8081/subjects/kcmi-CAR_STREAM_AVRO-value/versions/1
```
