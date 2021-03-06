# Debezium Connect Docker Image

A Debezium Docker Image that is entirely configurable via **environment variables**; It removes the need to manually POST connector configs to Connect's REST API. 

[View on Docker Hub](https://hub.docker.com/r/dhet/debezium-connect)

## Example
```
docker run --rm -ti dhet/debezium-connect \
  -e BOOTSTRAP_SERVERS=kafka:29092 \
  -e CONFIG_STORAGE_TOPIC=debezium-config \
  -e OFFSET_STORAGE_TOPIC=debezium-offset \
  -e STATUS_STORAGE_TOPIC=debezium-status \
  -e DEBEZIUM_CONFIG_CONNECTOR_CLASS=io.debezium.connector.postgresql.PostgresConnector \
  -e DEBEZIUM_CONFIG_DATABASE_HOSTNAME=database \
  -e DEBEZIUM_CONFIG_DATABASE_PORT=5432 \
  -e DEBEZIUM_CONFIG_DATABASE_PASSWORD=postgres \
  -e DEBEZIUM_CONFIG_DATABASE_USER=postgres \
  -e DEBEZIUM_CONFIG_DATABASE_SERVER_NAME=postgres \
  -e DEBEZIUM_CONFIG_DATABASE_DBNAME=postgres
```
See the [example folder](./example) for a Docker compose example. Running `docker-compose up` in that directory will spin up
* a single-broker Kafka setup, including Zookeeper
* a PostgreSQL database
* a Debezium connector that monitors the database and publishes changes to Kafka

## Configuration
This image adds a thin layer on top of the official [Debezium connect image](https://hub.docker.com/r/debezium/connect). As such, it supports all environment variables listed in the documentation, such as 

* `CONFIG_STORAGE_TOPIC` 
* `BOOTSTRAP_SERVERS`
* etc. 

### Configuring Connect
Similarly, all Kafka Connect configuration options are supported. As described in the docs they are prefixed with `CONNECT_` and the dots in the property name is translated into an underscore. For example

* `CONNECT_SSL_TRUSTSTORE_LOCATION` translates into `ssl.truststore.location=...`
* `CONNECT_KEY_CONVERTER` translates into `key.converter=...`
* etc.

### Configuring Debezium
In addition to the above, all configuration properties which are typically specified in a Debezium config file can be configured via environment variables. The translation scheme follows the example of Connect, except that the prefix is `DEBEZIUM_CONFIG_` *(this can be reconfigured via the `ENV_PREFIX` environment variable)*

Example:
* `DEBEZIUM_CONFIG_CONNECTOR_CLASS` translates into `connector.class=...`
* `DEBEZIUM_CONFIG_DATABASE_HOSTNAME` translates into `database.hostname=...`
* `DEBEZIUM_CONFIG_DATABASE_PORT` translates into `database.port=...`
* etc.

> ⚠️ The image is not capable of identifying invalid properties and is thus susceptible to typos. It simply translates the environment variables in accordance with the aforementioned renaming rules.

Additionally, the connector's name can be specified via `CONNECTOR_NAME` env var (the default connector name is "debezium")

## How it works
Setting up a Kafka connector involves sending an HTTP request to the connector's REST API which is often done as a manual step. The API takes the config and stores it in a log-compacted Kafka topic. Connectors are therefore inherently stateful and their deployment is a pain to automate.

This image tries to alleviate this by making the connector configurable through environment variables. When the container starts it builds a config file from the specified environment variables and PUTs it to the API. This is an idempotent operation=the first request will set up a new connector, and all subsequent requests will overwrite the existing configuration. As a result, starting the container several times with the same configuration will always have the same effect.

> ⚠️ The image is not capable of deleting existing configurations so a manual cleanup via [HTTP DELETE](https://docs.confluent.io/current/connect/references/restapi.html#delete--connectors-(string-name)-) is necessary in case Debezium is not needed anymore. 

