[![CircleCI](https://circleci.com/gh/Apicurio/apicurio-registry.svg?style=svg)](https://circleci.com/gh/Apicurio/apicurio-registry)

# Apicurio Registry

An API/Schema registry - stores and retrieves APIs and Schemas.

## Build Configuration

This project supports several build configuration options that affect the produced executables.

By default, `mvn clean install` produces an executable JAR with the *dev* Quarkus configuration profile enabled, and *in-memory* persistence implementation. 

Apicurio Registry supports 3 persistence implementations:
 - in-memory
 - Kafka
 - JPA.
 
If you enable one, a separate set of artifacts is produced with the persistence implementation available.

Additionally, there are 2 main configuration profiles:
 - *dev* - suitable for development, and
 - *prod* - for production environment.

### Build Options
 
 - `-Pkafka` enables a build of `storage/kafka` module and produces `apicurio-registry-storage-kafka-<version>-all.zip`.
 - `-Pjpa` enables a build of `storage/jpa` module and produces `apicurio-registry-storage-jpa-<version>-all.zip`. This artifact uses `H2` driver in *dev* mode,
   and `PostgreSQL` driver in *prod* mode.
 - `-Pprod` enables Quarkus's *prod* configuration profile, which uses configuration options suitable for a production environment, 
   e.g. a higher logging level.
 - `-Pnative` *(experimental)* builds native executables. See [Building a native executable](https://quarkus.io/guides/maven-tooling#building-a-native-executable). 
 - `-Ddocker` *(experimental)* builds docker images. Make sure that you have the docker service enabled and running.
   If you get an error, try `sudo chmod a+rw /var/run/docker.sock`.

## Runtime Configuration

The following parameters are available for executable files:

### JPA
 - In the *dev* mode, the application expects a H2 server running at `jdbc:h2:tcp://localhost:9123/mem:registry`.
 - In the *prod* mode, you have to provide connection configuration for a PostgreSQL server as follows:
  
|Option|Command argument|Env. variable|
|---|---|---|
|Data Source URL|`-Dquarkus.datasource.url`|`QUARKUS_DATASOURCE_URL`|
|DS Username|`-Dquarkus.datasource.username`|`QUARKUS_DATASOURCE_USERNAME`|
|DS Password|`-Dquarkus.datasource.password`|`QUARKUS_DATASOURCE_PASSWORD`|

To see additional options, visit:
 - [Data Source options](https://quarkus.io/guides/datasource-guide#configuration-reference) 
 - [Hibernate options](https://quarkus.io/guides/hibernate-orm-guide#properties-to-refine-your-hibernate-orm-configuration)
    
### Kafka

 - In the *dev* mode, the application expects a Kafka broker running at `localhost:9092`.
 - In the *prod* mode, you have to provide an environment variable KAFKA_BOOTSTRAP_SERVERS pointing to Kafka brokers

Kafka storage implementation uses the following Kafka API / architecture

 - Storage producer to forward REST API HTTP requests to Kafka broker
 - Storage consumer to handle previously sent  REST API HTTP requests as Kafka messages
 - Snapshot producer to send current state's snapshot to Kafka broker
 - Snapshot consumer for initial (at application start) snapshot handling

We already have sensible defaults for all these things, but they can still be overridden or added by adding appropriate properties to app's configuration. The following property name prefix must be used:

 - Storage producer: registry.kafka.storage-producer.
 - Storage consumer: registry.kafka.storage-consumer.
 - Snapshot producer: registry.kafka.snapshot-producer.
 - Snapshot consumer: registry.kafka.snapshot-consumer.

We then strip away the prefix and use the rest of the property name in instance's Properties.

e.g. registry.kafka.storage-producer.enable.idempotence=true --> enable.idempotence=true

For the actual configuration options check (although best config docs are in the code itself):
 - [Kafka configuration](https://kafka.apache.org/documentation/)

To help setup development / testing environment for the module, see kafka_setup.sh script. You just need to have KAFKA_HOME env variable set, and script does the rest.

### Streams

Streams storage implementation goes beyond plain Kafka usage and uses Kafka Streams to handle storage in a distributed and fault-tolerant way.

 - In the *dev* mode, the application expects a Kafka broker running at `localhost:9092`.
 - In the *prod* mode, you have to provide an environment variable KAFKA_BOOTSTRAP_SERVERS pointing to Kafka brokers and APPLICATION_ID to name your Kafka Streams application

Both modes require 2 topics: storage topic and globalId topic. This is configurable, by default we use storage-topic and global-id-topic names.

Streams storage implementation uses the following Kafka (Streams) API / architecture

 - Storage producer to forward REST API HTTP requests to Kafka broker
 - Streams input KStream to handle previously sent  REST API HTTP requests as Kafka messages
 - Streams KStream to handle input's KStream result
 - Both KStreams use KeyValueStores to keep current storage state

The two KeyValueStores keep the following structure:
 - storage store: <String, Str.Data> -- where the String is artifactId and Str.Data is whole artifact info: content, metadata, rules, etc
 - global id store: <Long, Str.Tuple> -- where the Long is unique globalId and Str.Tuple is a <artifactId, version> pair
 
We use global id store to map unique id to <artifactId, version> pair, which also uniquely identifies an artifact.
The data is distributed among node's stores, where we access those remote stores based on key distribution via gRPC. 

We already have sensible defaults for all these things, but they can still be overridden or added by adding appropriate properties to app's configuration. The following property name prefix must be used:

 - Storage producer: registry.streams.storage-producer.
 - Streams topology: registry.streams.topology.

We then strip away the prefix and use the rest of the property name in instance's Properties.

e.g. registry.streams.topology.replication.factor=1 --> replication.factor=1

For the actual configuration options check (although best config docs are in the code itself):
 - [Kafka configuration](https://kafka.apache.org/documentation/)
 - [Kafka Streams](https://kafka.apache.org/documentation/streams/)

To help setup development / testing environment for the module, see streams_setup.sh script. You just need to have KAFKA_HOME env variable set, and script does the rest.

### Docker container
The same options are available for the docker containers, but only in the form of environment variables (The command line parameters are for the `java` executable and at the moment it's not possible to pass them into the container).

## Examples

Run Apicurio Registry with Postgres:

 - Compile using `mvn clean install -DskipTests -Pprod -Pjpa -Ddocker`

 - Then create a docker-compose file `test.yml`: 
```yaml
version: '3.1'

services:
  postgres:
    image: postgres
    environment:
      POSTGRES_USER: apicurio-registry
      POSTGRES_PASSWORD: password
  app:
    image: apicurio/apicurio-registry-jpa:1.0.0-SNAPSHOT
    ports:
      - 8080:8080
    environment:
      QUARKUS_DATASOURCE_URL: 'jdbc:postgresql://postgres/apicurio-registry'
      QUARKUS_DATASOURCE_USERNAME: apicurio-registry
      QUARKUS_DATASOURCE_PASSWORD: password
```
  - Run `docker-compose -f test.yml up`

