# 2023-scyllaDb-CDC-integration

## Build a CDC integration between ScyllaDB and Redpanda
To get started with the setup, first, you need to go through several prerequisites, and then you’ll be able to configure CDC that will read data from ScyllaDB into Redpanda.
### Prerequisites

To complete this tutorial you'll need to have all of the following:

* [Docker Engine](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/)
* A [Redpanda instance on Docker](https://docs.redpanda.com/docs/get-started/quick-start/?quickstart=docker)
* A [ScyllaDB instance running on Docker](https://hub.docker.com/r/scylladb/scylla-enterprise)
* [ScyllaDB Kafka Connect drivers](https://docs.scylladb.com/stable/using-scylla/integrations/scylla-cdc-source-connector-quickstart.html)
* The [`jq` CLI](https://stedolan.github.io/jq/) tool
* The [`rpk` CLI](https://docs.redpanda.com/docs/get-started/rpk-install/) tool

> Note: The operating system used in this tutorial is macOS.

### Set Up Streaming between ScyllaDB and Redpanda Using CDC

After setting up ScyllaDB and Redpanda, you have to start a Standalone Kafka Connect cluster and use the ScyllaDB CDC Connector JAR files as plugins. Once you do that, a connection will be established between ScyllaDB and Kafka Connect. This tutorial uses [ScyllaDB's CDC Quickstart tutorial](https://docs.scylladb.com/stable/using-scylla/integrations/scylla-cdc-source-connector-quickstart.html#configuration-using-open-source-kafka) that takes an arbitrary `orders` table for an e-commerce business and streams data from that to a sink using the Kafka Connect-compatible Debezium connector. The following image depicts the simplified architecture of the setup:

![Connecting ScyllaDB and Redpanda using Kafka Connect - Image by author](https://i.ibb.co/C5GZdxr/upload-ac0f91072aac1177236f8ec64cd39b91-2.png)

The `orders` table receives new orders and updates on the previous orders. In this example, you'll insert a few simple orders with an `order_id`, a `customer_id`, and a `product`. You'll first insert a few records in the `orders` table and then perform a change on one of the records. All the data, including new records, changes, and deletes, will be available as change events on the Redpanda topic you've tied to the ScyllaDB `orders` table.

### Running and Configuring ScyllaDB

After installing and starting Docker Engine on your machine, execute the following Docker command to get ScyllaDB up and running:

```shell
docker run --rm -ti \
  -p 127.0.0.1:9042:9042 scylladb/scylla \
  --smp 1 --listen-address 0.0.0.0 \
  --broadcast-rpc-address 127.0.0.1
```

This command will spin up a container with ScyllaDB, accessible on 127.0.0.1 on port 9042. To check ScyllaDB's status, run the following command that uses the [ScyllaDB nodetool](https://docs.scylladb.com/stable/operating-scylla/nodetool.html):

```shell
docker exec -it 225a2369a71f nodetool status 
```

This command should give you an output that looks something like the following:

```shell
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address  Load       Tokens       Owns    Host ID                               Rack
UN  0.0.0.0  256 KB     256          ?       2597950d-9cc6-47eb-b3d6-a54076860321  rack1
```

The `UN` at the beginning of the table output means `Up` and `Normal`. These two represent the cluster's `status` and `state`, respectively.

#### Configuring CDC on ScyllaDB

To set up CDC on ScyllaDB, you first need to log into the cluster using the `cqlsh` CLI. You can do that using the following command:

```shell
docker exec -it 225a2369a71f cqlsh
```

If you are able to log in successfully, you'll see the following message:

```shell
Connected to  at 0.0.0.0:9042.
[cqlsh 5.0.1 | Cassandra 3.0.8 | CQL spec 3.3.1 | Native protocol v4]
Use HELP for help.
cqlsh>
```
Keyspaces are high-level containers of all the data in ScyllaDB. A keyspace in ScyllaDB is conceptually similar to a schema or database in MySQL. Use the following command to create a new keyspace called `quickstart_keyspace` with `SimpleStrategy` replication with a `replication_factor` of 1:
```sql
CREATE KEYSPACE quickstart_keyspace WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 1};
```

Now, use this keyspace for further CQL commands.

```sql
USE quickstart_keyspace;
```

Please note that the `SimpleStrategy` replication class is not recommended for production use. Instead, you should use the `NetworkTopologyStrategy` replication class. Learn more about replication methodologies from [ScyllaDB University](https://university.scylladb.com/courses/scylla-essentials-overview/lessons/architecture/topic/replication-strategy/).

#### Creating the `orders` Table

Using the following CQL statement to create the `orders` table described at the beginning of the tutorial:

```sql
CREATE TABLE orders(
   customer_id int,
   order_id int,
   product text,
   PRIMARY KEY(customer_id, order_id)) WITH cdc = {'enabled': true};
```

This statement creates the `orders` table with the composite primary key consisting of `customer_id` and `order_id`. The `orders` table will be CDC-enabled.

ScyllaDB stores table data in a base table, but when you enable CDC on that table, an additional table that captures all the changed data is created too. That table is called a [log table](https://docs.scylladb.com/stable/using-scylla/cdc/cdc-log-table.html).

#### Insert a Few Initial Records

Populate a few records for testing purposes:

```sql
INSERT INTO quickstart_keyspace.orders(customer_id, order_id, product) VALUES (1, 1, 'pizza');
INSERT INTO quickstart_keyspace.orders(customer_id, order_id, product) VALUES (1, 2, 'cookies');
INSERT INTO quickstart_keyspace.orders(customer_id, order_id, product) VALUES (1, 3, 'tea');
```

During the course of the tutorial, you'll insert three more records and perform an update on one record, totaling seven events in total, four out of which are change events after the initial setup.

### Setting up Redpanda

Use the [`docker-compose.yaml`](https://github.com/redpanda-data-blog/2023-scyllaDb-CDC-integration/blob/main/docker-compose.yaml) file in the [Redpanda Docker quickstart tutorial](https://docs.redpanda.com/docs/get-started/quick-start/?quickstart=docker) to run the following command:

```shell
docker compose up -d
```

A Redpanda cluster and a Redpanda console will be up and running after a brief wait. You can check the status of the Redpanda cluster using the following command:


```shell
docker exec -it redpanda-0 rpk cluster info
```

The output of the command should look something like the following:

```shell
CLUSTER
=======
redpanda.3fdc3646-9c9d-4eff-b5d6-854093a25b67

BROKERS
=======
ID    HOST        PORT
0*    redpanda-0  9092
```

Before setting up an integration between ScyllaDB and Redpanda, check if all the Docker containers you have spawned are running using the following command:

```shell
docker ps --format "table {{.Image}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Look for the `STATUS` column in the table output:

```shell
IMAGE                                               NAMES              STATUS       PORTS
scylladb/scylla                                     gifted_hertz       Up 8 hours   22/tcp, 7000-7001/tcp, 9160/tcp, 9180/tcp, 10000/tcp, 127.0.0.1:9042->9042/tcp
docker.redpanda.com/vectorized/console:v2.2.4       redpanda-console   Up 9 hours   0.0.0.0:8080->8080/tcp
docker.redpanda.com/redpandadata/redpanda:v23.1.8   redpanda-0         Up 9 hours   8081-8082/tcp, 0.0.0.0:18081-18082->18081-18082/tcp, 9092/tcp, 0.0.0.0:19092->19092/tcp, 0.0.0.0:19644->9644/tcp
```

If these containers are up and running, you can start setting up the integration.



### Setting up Kafka Connect

Download and install Kafka using the following sequence of commands:

```shell
# Extract the binaries
wget https://downloads.apache.org/kafka/3.4.0/kafka_2.13-3.4.0.tgz && tar -xzf kafka_2.13-3.4.0.tgz && cd kafka_2.13-3.4.0

#start Kafka connect in standalone mode 
bin/connect-standalone.sh config/connect-standalone.properties
```

The properties files will be kept in the `/Users/kovidrathee/Downloads/kafka_2.13-3.4.0/config` directory.

To use Kafka Connect, you must configure and validate two properties files. You can name these files anything. In this tutorial, the two files are called `connect-standalone.properties` and `connector.properties`. The first file being the one that contains the properties for the standalone Kafka Connect instance. For this file, you will only change the default value of two variables:

* `bootstrap.servers`: The default value for the `bootstrap.servers` variable is `localhost:9092`. As you're using Redpanda, whose broker is running on the port 19092, you'll replace the default value with `localhost:19092`.
* `plugin.path`: Find the plugin path directory for your Kafka installation and set this variable to that path. In this case, it is `/usr/local/share/kafka/plugins`. This is where your [ScyllaDB CDC Connector JAR file](https://github.com/scylladb/scylla-cdc-source-connector#building) will be copied.

The comment-stripped version of the `connect-standalone.properties` file should look like the following:

```java
bootstrap.servers=localhost:19092
key.converter.schemas.enable=true
value.converter.schemas.enable=true
offset.storage.file.filename=/tmp/connect.offsets
offset.flush.interval.ms=10000
plugin.path=/usr/local/share/kafka/plugins
```

The second properties file will contain settings specific to the ScyllaDB connector. You need to change the `scylla.cluster.ip.addresses` variable to `127.0.0.1:9042`. After that change, the `connector.properties` file should look like the following:

```java
name = QuickstartConnector
connector.class = com.scylladb.cdc.debezium.connector.ScyllaConnector
key.converter = org.apache.kafka.connect.json.JsonConverter
value.converter = org.apache.kafka.connect.json.JsonConverter
scylla.cluster.ip.addresses = 127.0.0.1:9042
scylla.name = QuickstartConnectorNamespace
scylla.table.names = quickstart_keyspace.orders
```

Using both the properties files, go to the Kafka installation directory and run the `connect-standalone.sh` script with the following command:

```shell
bin/connect-standalone.sh config/connect-standalone.properties config/connector.properties
```

When you created the `orders` table, you enabled CDC, which means that there's a log table with all the records and changes. If the Kafka Connect setup is successful, you should now be able to consume these events using the `rpk` CLI tool using the following command:

```shell
rpk topic consume --brokers 'localhost:19092' QuickstartConnectorNamespace.quickstart_keyspace.orders | jq .
```

The output should result in the following three records, as shown below:

```json
{
  "topic": "QuickstartConnectorNamespace.quickstart_keyspace.orders",
  "key": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Key\"},\"payload\":{\"customer_id\":1,\"order_id\":1}}",
  "value": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"version\"},{\"type\":\"string\",\"optional\":false,\"field\":\"connector\"},{\"type\":\"string\",\"optional\":false,\"field\":\"name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_ms\"},{\"type\":\"string\",\"optional\":true,\"name\":\"io.debezium.data.Enum\",\"version\":1,\"parameters\":{\"allowed\":\"true,last,false\"},\"default\":\"false\",\"field\":\"snapshot\"},{\"type\":\"string\",\"optional\":false,\"field\":\"db\"},{\"type\":\"string\",\"optional\":false,\"field\":\"keyspace_name\"},{\"type\":\"string\",\"optional\":false,\"field\":\"table_name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_us\"}],\"optional\":false,\"name\":\"com.scylladb.cdc.debezium.connector\",\"field\":\"source\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Before\",\"field\":\"before\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.After\",\"field\":\"after\"},{\"type\":\"string\",\"optional\":true,\"field\":\"op\"},{\"type\":\"int64\",\"optional\":true,\"field\":\"ts_ms\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"id\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"total_order\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"data_collection_order\"}],\"optional\":true,\"field\":\"transaction\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Envelope\"},\"payload\":{\"source\":{\"version\":\"1.0.1\",\"connector\":\"scylla\",\"name\":\"QuickstartConnectorNamespace\",\"ts_ms\":1683357282912,\"snapshot\":\"false\",\"db\":\"quickstart_keyspace\",\"keyspace_name\":\"quickstart_keyspace\",\"table_name\":\"orders\",\"ts_us\":1683357282912753},\"before\":null,\"after\":{\"customer_id\":1,\"order_id\":1,\"product\":{\"value\":\"pizza\"}},\"op\":\"c\",\"ts_ms\":1683357426553,\"transaction\":null}}",
  "timestamp": 1683357426891,
  "partition": 0,
  "offset": 0
}
{
  "topic": "QuickstartConnectorNamespace.quickstart_keyspace.orders",
  "key": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Key\"},\"payload\":{\"customer_id\":1,\"order_id\":2}}",
  "value": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"version\"},{\"type\":\"string\",\"optional\":false,\"field\":\"connector\"},{\"type\":\"string\",\"optional\":false,\"field\":\"name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_ms\"},{\"type\":\"string\",\"optional\":true,\"name\":\"io.debezium.data.Enum\",\"version\":1,\"parameters\":{\"allowed\":\"true,last,false\"},\"default\":\"false\",\"field\":\"snapshot\"},{\"type\":\"string\",\"optional\":false,\"field\":\"db\"},{\"type\":\"string\",\"optional\":false,\"field\":\"keyspace_name\"},{\"type\":\"string\",\"optional\":false,\"field\":\"table_name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_us\"}],\"optional\":false,\"name\":\"com.scylladb.cdc.debezium.connector\",\"field\":\"source\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Before\",\"field\":\"before\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.After\",\"field\":\"after\"},{\"type\":\"string\",\"optional\":true,\"field\":\"op\"},{\"type\":\"int64\",\"optional\":true,\"field\":\"ts_ms\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"id\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"total_order\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"data_collection_order\"}],\"optional\":true,\"field\":\"transaction\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Envelope\"},\"payload\":{\"source\":{\"version\":\"1.0.1\",\"connector\":\"scylla\",\"name\":\"QuickstartConnectorNamespace\",\"ts_ms\":1683357282913,\"snapshot\":\"false\",\"db\":\"quickstart_keyspace\",\"keyspace_name\":\"quickstart_keyspace\",\"table_name\":\"orders\",\"ts_us\":1683357282913843},\"before\":null,\"after\":{\"customer_id\":1,\"order_id\":2,\"product\":{\"value\":\"cookies\"}},\"op\":\"c\",\"ts_ms\":1683357426566,\"transaction\":null}}",
  "timestamp": 1683357426898,
  "partition": 0,
  "offset": 1
}
{
  "topic": "QuickstartConnectorNamespace.quickstart_keyspace.orders",
  "key": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Key\"},\"payload\":{\"customer_id\":1,\"order_id\":3}}",
  "value": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"version\"},{\"type\":\"string\",\"optional\":false,\"field\":\"connector\"},{\"type\":\"string\",\"optional\":false,\"field\":\"name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_ms\"},{\"type\":\"string\",\"optional\":true,\"name\":\"io.debezium.data.Enum\",\"version\":1,\"parameters\":{\"allowed\":\"true,last,false\"},\"default\":\"false\",\"field\":\"snapshot\"},{\"type\":\"string\",\"optional\":false,\"field\":\"db\"},{\"type\":\"string\",\"optional\":false,\"field\":\"keyspace_name\"},{\"type\":\"string\",\"optional\":false,\"field\":\"table_name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_us\"}],\"optional\":false,\"name\":\"com.scylladb.cdc.debezium.connector\",\"field\":\"source\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Before\",\"field\":\"before\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.After\",\"field\":\"after\"},{\"type\":\"string\",\"optional\":true,\"field\":\"op\"},{\"type\":\"int64\",\"optional\":true,\"field\":\"ts_ms\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"id\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"total_order\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"data_collection_order\"}],\"optional\":true,\"field\":\"transaction\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Envelope\"},\"payload\":{\"source\":{\"version\":\"1.0.1\",\"connector\":\"scylla\",\"name\":\"QuickstartConnectorNamespace\",\"ts_ms\":1683357282914,\"snapshot\":\"false\",\"db\":\"quickstart_keyspace\",\"keyspace_name\":\"quickstart_keyspace\",\"table_name\":\"orders\",\"ts_us\":1683357282914266},\"before\":null,\"after\":{\"customer_id\":1,\"order_id\":3,\"product\":{\"value\":\"tea\"}},\"op\":\"c\",\"ts_ms\":1683357426594,\"transaction\":null}}",
  "timestamp": 1683357426898,
  "partition": 0,
  "offset": 2
}
```

If you get a similar output, you've successfully integrated ScyllaDB with Redpanda using Kafka Connect. Alternatively, you can go to the Redpanda console hosted on `localhost:8080` and see if the topic corresponding to the ScyllaDB `orders` table is available:

![A working connection between ScyllaDB and Redpanda on the Redpanda console](https://i.ibb.co/RDCTc43/upload-02c999eb591a9823fff75389cf648087-1.png)

You can now test whether data changes to the `orders` table can trigger CDC.

### Capturing Change Data from ScyllaDB

To test CDC for new records, insert the following two records in the `orders` table using the `cqlsh` CLI:

```sql
INSERT INTO quickstart_keyspace.orders(customer_id, order_id, product) VALUES (1, 4, 'chips');
INSERT INTO quickstart_keyspace.orders(customer_id, order_id, product) VALUES (1, 5, 'lollies');
```

If the insert is successful, the `rpk topic consume` command will give you the following additional records:

```json
{
  "topic": "QuickstartConnectorNamespace.quickstart_keyspace.orders",
  "key": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Key\"},\"payload\":{\"customer_id\":1,\"order_id\":4}}",
  "value": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"version\"},{\"type\":\"string\",\"optional\":false,\"field\":\"connector\"},{\"type\":\"string\",\"optional\":false,\"field\":\"name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_ms\"},{\"type\":\"string\",\"optional\":true,\"name\":\"io.debezium.data.Enum\",\"version\":1,\"parameters\":{\"allowed\":\"true,last,false\"},\"default\":\"false\",\"field\":\"snapshot\"},{\"type\":\"string\",\"optional\":false,\"field\":\"db\"},{\"type\":\"string\",\"optional\":false,\"field\":\"keyspace_name\"},{\"type\":\"string\",\"optional\":false,\"field\":\"table_name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_us\"}],\"optional\":false,\"name\":\"com.scylladb.cdc.debezium.connector\",\"field\":\"source\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Before\",\"field\":\"before\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.After\",\"field\":\"after\"},{\"type\":\"string\",\"optional\":true,\"field\":\"op\"},{\"type\":\"int64\",\"optional\":true,\"field\":\"ts_ms\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"id\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"total_order\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"data_collection_order\"}],\"optional\":true,\"field\":\"transaction\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Envelope\"},\"payload\":{\"source\":{\"version\":\"1.0.1\",\"connector\":\"scylla\",\"name\":\"QuickstartConnectorNamespace\",\"ts_ms\":1683357710933,\"snapshot\":\"false\",\"db\":\"quickstart_keyspace\",\"keyspace_name\":\"quickstart_keyspace\",\"table_name\":\"orders\",\"ts_us\":1683357710933359},\"before\":null,\"after\":{\"customer_id\":1,\"order_id\":4,\"product\":{\"value\":\"chips\"}},\"op\":\"c\",\"ts_ms\":1683357768114,\"transaction\":null}}",
  "timestamp": 1683357768358,
  "partition": 0,
  "offset": 3
}
{
  "topic": "QuickstartConnectorNamespace.quickstart_keyspace.orders",
  "key": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Key\"},\"payload\":{\"customer_id\":1,\"order_id\":5}}",
  "value": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"version\"},{\"type\":\"string\",\"optional\":false,\"field\":\"connector\"},{\"type\":\"string\",\"optional\":false,\"field\":\"name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_ms\"},{\"type\":\"string\",\"optional\":true,\"name\":\"io.debezium.data.Enum\",\"version\":1,\"parameters\":{\"allowed\":\"true,last,false\"},\"default\":\"false\",\"field\":\"snapshot\"},{\"type\":\"string\",\"optional\":false,\"field\":\"db\"},{\"type\":\"string\",\"optional\":false,\"field\":\"keyspace_name\"},{\"type\":\"string\",\"optional\":false,\"field\":\"table_name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_us\"}],\"optional\":false,\"name\":\"com.scylladb.cdc.debezium.connector\",\"field\":\"source\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Before\",\"field\":\"before\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.After\",\"field\":\"after\"},{\"type\":\"string\",\"optional\":true,\"field\":\"op\"},{\"type\":\"int64\",\"optional\":true,\"field\":\"ts_ms\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"id\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"total_order\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"data_collection_order\"}],\"optional\":true,\"field\":\"transaction\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Envelope\"},\"payload\":{\"source\":{\"version\":\"1.0.1\",\"connector\":\"scylla\",\"name\":\"QuickstartConnectorNamespace\",\"ts_ms\":1683358016676,\"snapshot\":\"false\",\"db\":\"quickstart_keyspace\",\"keyspace_name\":\"quickstart_keyspace\",\"table_name\":\"orders\",\"ts_us\":1683358016676506},\"before\":null,\"after\":{\"customer_id\":1,\"order_id\":5,\"product\":{\"value\":\"lollies\"}},\"op\":\"c\",\"ts_ms\":1683358068107,\"transaction\":null}}",
  "timestamp": 1683358068355,
  "partition": 0,
  "offset": 4
}
```

You'll now insert one more record with the `product` value `pasta`. Later on, you'll change this value with an `UPDATE` statement to `spaghetti` and trigger an CDC update event.

```sql
INSERT INTO quickstart_keyspace.orders(customer_id, order_id, product) VALUES (1, 5, 'pasta');
```

The newly inserted record should be visible with your `rpk topic consume` command:

```json
{
  "topic": "QuickstartConnectorNamespace.quickstart_keyspace.orders",
  "key": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Key\"},\"payload\":{\"customer_id\":1,\"order_id\":6}}",
  "value": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"version\"},{\"type\":\"string\",\"optional\":false,\"field\":\"connector\"},{\"type\":\"string\",\"optional\":false,\"field\":\"name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_ms\"},{\"type\":\"string\",\"optional\":true,\"name\":\"io.debezium.data.Enum\",\"version\":1,\"parameters\":{\"allowed\":\"true,last,false\"},\"default\":\"false\",\"field\":\"snapshot\"},{\"type\":\"string\",\"optional\":false,\"field\":\"db\"},{\"type\":\"string\",\"optional\":false,\"field\":\"keyspace_name\"},{\"type\":\"string\",\"optional\":false,\"field\":\"table_name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_us\"}],\"optional\":false,\"name\":\"com.scylladb.cdc.debezium.connector\",\"field\":\"source\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Before\",\"field\":\"before\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.After\",\"field\":\"after\"},{\"type\":\"string\",\"optional\":true,\"field\":\"op\"},{\"type\":\"int64\",\"optional\":true,\"field\":\"ts_ms\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"id\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"total_order\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"data_collection_order\"}],\"optional\":true,\"field\":\"transaction\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Envelope\"},\"payload\":{\"source\":{\"version\":\"1.0.1\",\"connector\":\"scylla\",\"name\":\"QuickstartConnectorNamespace\",\"ts_ms\":1683361126204,\"snapshot\":\"false\",\"db\":\"quickstart_keyspace\",\"keyspace_name\":\"quickstart_keyspace\",\"table_name\":\"orders\",\"ts_us\":1683361126204989},\"before\":null,\"after\":{\"customer_id\":1,\"order_id\":6,\"product\":{\"value\":\"pasta\"}},\"op\":\"c\",\"ts_ms\":1683361158114,\"transaction\":null}}",
  "timestamp": 1683361158363,
  "partition": 0,
  "offset": 5
}
```

Now, execute the following update statement and see if a CDC update event is triggered:

```sql
UPDATE quickstart_keyspace.orders SET product = 'spaghetti' WHERE order_id = 6 and customer_id = 1;
```

After running this command, you'll need to run the `rpk topic consume` command to verify the latest addition to the `QuickstartConnectorNamespace.quickstart_keyspace.orders` topic. The change event record should look like the following:

```json
{
  "topic": "QuickstartConnectorNamespace.quickstart_keyspace.orders",
  "key": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Key\"},\"payload\":{\"customer_id\":1,\"order_id\":6}}",
  "value": "{\"schema\":{\"type\":\"struct\",\"fields\":[{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"version\"},{\"type\":\"string\",\"optional\":false,\"field\":\"connector\"},{\"type\":\"string\",\"optional\":false,\"field\":\"name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_ms\"},{\"type\":\"string\",\"optional\":true,\"name\":\"io.debezium.data.Enum\",\"version\":1,\"parameters\":{\"allowed\":\"true,last,false\"},\"default\":\"false\",\"field\":\"snapshot\"},{\"type\":\"string\",\"optional\":false,\"field\":\"db\"},{\"type\":\"string\",\"optional\":false,\"field\":\"keyspace_name\"},{\"type\":\"string\",\"optional\":false,\"field\":\"table_name\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"ts_us\"}],\"optional\":false,\"name\":\"com.scylladb.cdc.debezium.connector\",\"field\":\"source\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Before\",\"field\":\"before\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"int32\",\"optional\":true,\"field\":\"customer_id\"},{\"type\":\"int32\",\"optional\":true,\"field\":\"order_id\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":true,\"field\":\"value\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.product.Cell\",\"field\":\"product\"}],\"optional\":true,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.After\",\"field\":\"after\"},{\"type\":\"string\",\"optional\":true,\"field\":\"op\"},{\"type\":\"int64\",\"optional\":true,\"field\":\"ts_ms\"},{\"type\":\"struct\",\"fields\":[{\"type\":\"string\",\"optional\":false,\"field\":\"id\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"total_order\"},{\"type\":\"int64\",\"optional\":false,\"field\":\"data_collection_order\"}],\"optional\":true,\"field\":\"transaction\"}],\"optional\":false,\"name\":\"QuickstartConnectorNamespace.quickstart_keyspace.orders.Envelope\"},\"payload\":{\"source\":{\"version\":\"1.0.1\",\"connector\":\"scylla\",\"name\":\"QuickstartConnectorNamespace\",\"ts_ms\":1683362835504,\"snapshot\":\"false\",\"db\":\"quickstart_keyspace\",\"keyspace_name\":\"quickstart_keyspace\",\"table_name\":\"orders\",\"ts_us\":1683362835504855},\"before\":null,\"after\":{\"customer_id\":1,\"order_id\":6,\"product\":{\"value\":\"spaghetti\"}},\"op\":\"u\",\"ts_ms\":1683362868120,\"transaction\":null}}",
  "timestamp": 1683362868372,
  "partition": 0,
  "offset": 6
}
```
