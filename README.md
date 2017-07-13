## fuse-openshift-debezium-tutorial

Port of [Debezium tutorial](http://debezium.io/docs/tutorial/) on OpenShift using a JBoss Fuse Integration Service consumer sample

### Pre-requisites

It is assumed that you have some kind of OpenShift cluster instance running and available. This instance can take several forms depending on your environment and needs :
* Full blown OpenShift cluster at your site, see how to [Install OpenShift at your site](https://docs.openshift.com/container-platform/3.5/install_config/index.html),
* Red Hat Container Development Kit on your laptop, see how to [Get Started with CDK](http://developers.redhat.com/products/cdk/get-started/),
* Lightweight Minishift on your laptop, see [Minishift project page](https://github.com/minishift/minishift).

You should also have the `oc` client line interface tool installed on your machine. Pick the corresponding OpenShift version from [this page](https://github.com/openshift/origin/releases).

### Setup of tutorial

First, start by creating a dedicated `debezium`project for running this tutorial on your favourite OpenShift instance. You'll have to ensure to be in this project for the rest of tutorial.

```
oc new-project debezium --display-name=”Debezium” --description=”Debezium on Openshift”

```

Because MySQL tutorial image used by [Debezium tutorial](http://debezium.io/docs/tutorial/) needs `root` privileges and that is not allowed by default on OpenShift, you'll need to extend the privileges of default service account into your project. Use this :

```
oc adm policy add-scc-to-user anyuid -z default
```

Now, you 2 options :

#### Manual Installation

You have to setup all containers like described in the original [tutorial](http://debezium.io/docs/tutorial/).

However, you will need to :
* add the `KAFKA_ADVERTISED_HOSTNAME=kafka`, `ZOOKEEPER_CONNECT=zookeeper:2181` and `KAFKA_ADVERTISED_HOST_PORT=9092` environment variable for `kafka` deployment,
* add the `MYSQL_ROOT_PASSWORD=debezium`, `MYSQL_USER=mysqluser` and `MYSQL_PASSWORD=mysqlpw`environment variables for `mysql` deployment,
* ensure that `mysql` deployment service is called `mysql`,
* add the `GROUP_ID=1`, `CONFIG_STORAGE_TOPIC=my_connect_configs`, `OFFSET_STORAGE_TOPIC=my_connect_offsets`, `BOOTSTRAP_SERVERS=kafka:9092`, `ADVERTISED_HOST_NAME=connect` and `ADVERTISED_PORT=9092` environment variables to the `connect` deployment,
* add the `ZOOKEEPER_CONNECT=zookeeper:2181` environment variables to the `watcher` deployment. You'll also have to rename this deployment to `watcher`' so that it does not override the previously deployed `kafka` container,
* remove all the `volumeMounts`for `zookeeper`, `kafka` and `connect` deployment configuration created by OpenShift,
* create a `Route` for the `connect` service.

#### Automatic Installation

Just use the given template to create everything within your `debezium` OpenShift project and create a `Route` for easily accessing `connect` component later.

```
oc create -f openshift-template
oc expose svc/connect
```

### Setup of Debezium connector

Depending on the URL value for the route you have created for the `connect` component, you'all have to call it to declare a new connector to MySQL database :

```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://connect-debezium.192.168.99.100.nip.io/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'
```

Then, you can check connector existence :

```
curl -H "Accept:application/json" http://connect-debezium.192.168.99.100.nip.io/connectors/
```

And get connector details with this latest curl command :

```
curl -i -X GET -H "Accept:application/json" http://connect-debezium.192.168.99.100.nip.io/connectors/inventory-connector
```


### Building and deploying Fuse Integration Services project

The current project is a Fuse Integration Services project that just consumes Change Data Capture events emitted by Debezium from Kafka using different Camel routes. It just logs events, it's up to you to add your own transformation, business or synchronisation logics within the route.

Here's the description of the 3 routes :

```
<route id="customers-cdc" streamCache="true">
   <from id="from-customers" uri="kafka:kafka:9092?topic=dbserver1.inventory.customers&amp;groupId=2"/>
   <log id="log-customers" message="Customer change event: ${body}"/>
</route>

<route id="orders-cdc" streamCache="true">
   <from id="from-orders" uri="kafka:kafka:9092?topic=dbserver1.inventory.orders&amp;groupId=2"/>
   <log id="log-orders" message="Order change event: ${body}"/>
</route>

<route id="products-cdc" streamCache="true">
   <from id="from-products" uri="kafka:kafka:9092?topic=dbserver1.inventory.products&amp;groupId=2"/>
   <log id="log-products" message="Product change event: ${body}"/>
</route>
```

It is important noting here that :
* Each route is listening to a different topic created by Debezium and corresponding to changes on different tables of our database,
* We are using OpenShift/Kubernetes service discovery to be able to address the Kafka brokers using short notation (just `kafka:9200` and that's it),
* Kafka connectors should use a different `groupId` of the one used by Debezium (here we're using `2`).

To build it using Maven Fabric8 plugin, you'll have to setup your Docker environment variables. Using CDK or Minishift, it just ends up like this :

```
$ eval $(minishift docker-env)
$ oc project debezium
```

And then run Maven command to build and deploy everything :

```
$ mvn fabric8:deploy
```

This creates a new `dbz-tutorial` deployment config and associated pod with a Fuse container.

### Testing everything

In order to test if everything is fine and running as expected, here is a simple test procedure :
* open a terminal within the `mysql` container,
* follow logs on the `connect` container,
* follow logs on the `watcher` container,
* follow logs on the `dbz-tutorial` container.

In the `mysql` terminal, log to mysql console using `mysqluser` and `mysqlpw` on `inventory` database. Make a simple change on the `customers` table.

```
mysql -u mysqluser -p inventory                                                                                               

mysql> UPDATE customers SET first_name='Edouard' WHERE id=1003;                                                                                                                                                
Query OK, 1 row affected (0.00 sec)                                                                        
Rows matched: 1  Changed: 1  Warnings: 0
```

Check the `connect` logs that immediately reacts to the change in the database :

```
2017-07-13 09:16:34,172 INFO   MySQL|dbserver1|binlog  1 records sent during previous 00:15:51.998, last recorded offset: {ts_sec=1499937393, file=mysql-bin.000003, pos=580, row=1, server_id=223344, event=2}   [io.debezium.connector.mysql.BinlogReader]
2017-07-13 09:16:50,686 INFO   ||  Finished WorkerSourceTask{id=inventory-connector-0} commitOffsets successfully in 4 ms   [org.apache.kafka.connect.runtime.WorkerSourceTask]
```

Check the `watcher` logs that immediately prints the new event emitted by Debezium in Kafka :

```
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1003}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":{"id":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"},"after":{"id":1003,"first_name":"Edouard","last_name":"Walker","email":"ed@walker.com"},"source":{"name":"dbserver1","server_id":223344,"ts_sec":1499937393,"gtid":null,"file":"mysql-bin.000003","pos":725,"row":0,"snapshot":null,"thread":11,"db":"inventory","table":"customers"},
"op":"u","ts_ms":1499937393541}}
```

Check the `dbz-tutorial` logs that immediately prints the new event found by the Camel route using the Kafka connector :

```
09:16:34.265 [Camel (camel) thread #0 - KafkaConsumer[dbserver1.inventory.customers]] INFO  customers-cdc - Customer change event: {"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":{"id":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"},"after":{"id":1003,"first_name":"Edouard","last_name":"Walker","email":"ed@walker.com"},"source":{"name":"dbserver1","server_id":223344,"ts_sec":1499937393,"gtid":null,"file":"mysql-bin.000003","pos":725,"row":0,"snapshot":null,"thread":11,"db":"inventory","table":"customers"},"op":"u","ts_ms":1499937393541}}
```

Ta dam !
