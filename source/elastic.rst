Kafka Connect Elastic
=====================

A Connector and Sink to write events from Kafka to Elastic Search using `Elastic4s <https://github.com/sksamuel/elastic4s>`__ client.
The connector converts the value from the Kafka Connect SinkRecords to Json and uses Elastic4s's JSON insert functionality to index.

The Sink creates an Index and Type corresponding to the topic name and uses the JSON insert functionality from Elastic4s

Prerequisites
-------------

- Confluent 3.0.0
- Elastic Search 2.2
- Java 1.8
- Scala 2.11

Setup
-----

Elastic Setup
~~~~~~~~~~~~~

Download and start Elastic search.

.. sourcecode:: bash

    curl -L -O https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.2.0/elasticsearch-2.2.0.tar.gz
    tar -xvf elasticsearch-2.2.0.tar.gz
    cd elasticsearch-2.2.0/bin
    ./elasticsearch --cluster.name elasticsearch

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/3.0/confluent-3.0.0-2.11.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-3.0.0-2.11.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-3.0.0

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    bin/kafka-server-start etc/kafka/server.properties &
    bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from here and `here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    git clone https://github.com/datamountaineer/stream-reactor
    cd stream-reactor
    gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    git clone https://github.com/datamountaineer/kafka-connect-tools
    cd kafka-connect-tools
    gradle fatJar


Sink Connector QuickStart
-------------------------

Next we will start the connector in distributed mode. Connect has two modes, standalone where the tasks run on only one host
and distributed mode. Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode
you start Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are
distributed across the cluster.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a file called ``elastic-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=TOPIC1
    connect.cassandra.export.route.query=INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

This configuration defines:

1. The name of the connector.
2. The class containing the connector.
3. The Elastic Search URL.
4. Tne name of the cluster on the Elastic Search server to connect to.
5. The max number of task allowed for this connector.
6. The source topic to get records from.
7. The field mapping and topic to index routing.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

Now start the connector in distributed mode. We only give it one properties file for the kafka, zookeeper and
schema registry configurations.

First add the connector jar to the CLASSPATH and then start Connect.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-elastic-0.2-cp-3.0.0.all.jar

.. sourcecode:: bash

    ➜  confluent-3.0.0/bin/connect-distributed confluent-3.0.0/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.5-all.jar create elastic-sink < elastic-sink.properties

    #Connector name=`elastic-sink`
    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=TOPIC1
    connect.elastic.export.route.query=INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1
    #task ids: 0

If you switch back to the terminal you started the Connector in you should see the Elastic sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ java -jar build/libs/kafka-connect-cli-0.5-all.jar ps
    elastic-sink

.. sourcecode:: bash

    [2016-05-08 20:56:52,241] INFO

        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           ________           __  _      _____ _       __
          / ____/ /___ ______/ /_(_)____/ ___/(_)___  / /__
         / __/ / / __ `/ ___/ __/ / ___/\__ \/ / __ \/ //_/
        / /___/ / /_/ (__  ) /_/ / /__ ___/ / / / / / ,<
       /_____/_/\__,_/____/\__/_/\___//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:33)

    [2016-05-08 20:56:52,327] INFO [Hebe] loaded [], sites [] (org.elasticsearch.plugins:149)
    [2016-05-08 20:56:52,765] INFO Initialising Elastic Json writer (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:31)
    [2016-05-08 20:56:52,777] INFO Assigned List(test_table) topics. (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:33)
    [2016-05-08 20:56:52,836] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@69b6b39 finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)

Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic TOPIC1 \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"random_field", "type": "string"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"id": 999, "random_field": "foo"}
    {"id": 888, "random_field": "bar"}


Check for records in Elastic Search
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now if we check the logs of the connector we should see 2 records being inserted to Elastic Search:

.. sourcecode:: bash

    [2016-05-08 21:02:52,095] INFO Flushing Elastic Sink (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:73)
    [2016-05-08 21:03:52,097] INFO No records received. (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:63)
    [2016-05-08 21:03:52,097] INFO org.apache.kafka.connect.runtime.WorkerSinkTask@69b6b39 Committing offsets (org.apache.kafka.connect.runtime.WorkerSinkTask:187)
    [2016-05-08 21:03:52,097] INFO Flushing Elastic Sink (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:73)
    [2016-05-08 21:04:20,613] INFO Elastic write successful for 2 records! (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:77)

If we query Elastic Search for ``id`` 999:

.. sourcecode:: bash

    curl -XGET 'http://localhost:9200/INDEX_1/_search?q=id:999'

    {
        "took": 45,
        "timed_out": false,
        "_shards": {
            "total": 5,
            "successful": 5,
            "failed": 0
        },
        "hits": {
            "total": 1,
            "max_score": 1.2231436,
            "hits": [{
                "_index": "test_table",
                "_type": "test_table",
                "_id": "AVMY4eZXFguf2uMZyxjU",
                "_score": 1.2231436,
                "_source": {
                    "id": 999,
                    "random_field": "foo"
                }
            }]
        }
    }

Features
--------

1. Auto index creation at start up.
2. Topic to index mapping.
3. Auto mapping of the Kafka topic schema to the index.
4. Field selection

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`__
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Elastic sink supports the following:

.. sourcecode:: bash

    INSERT INTO <index> SELECT <fields> FROM <source topic>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to indexA
    INSERT INTO indexA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to indexB
    INSERT INTO indexB SELECT x AS a, y AS b and z AS c FROM topicB PK y

This is set in the ``connect.elastic.export.route.query`` option.

Auto Index Creation
~~~~~~~~~~~~~~~~~~~

The Sink will automatically create missing indexes at startup. The sink use elastic4s, more details can be found
`here <https://github.com/sksamuel/elastic4s>`__

Configurations
--------------

``connect.elastic.url``

Url of the Elastic cluster.

* Data Type : string
* Importance: high
* Optional  : no

``connect.elastic.port``

Port of the Elastic cluster.

* Data Type : string
* Importance: high
* Optional  : no

``connect.cassandra.export.route.query``

Kafka connect query language expression. Allows for expressive table to topic routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

* Data type : string
* Importance: high
* Optional  : no

Example
~~~~~~~

.. sourcecode:: bash

    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=test_table
    connect.cassandra.export.route.query=INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/2.0.1/schema-registry/docs/api.html#compatibility>`_.

Elastic Search is very flexible about what is inserted. All documents in Elasticsearch are stored in an index. We do not
need to tell Elasticsearch in advance what an index will look like (eg what fields it will contain) as Elasticsearch will
adapt the index dynamically as more documents are added, but we must at least create the index first. The Sink connector
automatically creates the index at start up if it doesn't exist.

The Elastic Search sink will automatically index if new fields are added to the source topic, if fields are removed
the Kafka Connect framework will return the default value for this field, dependent of the compatibility settings of the
Schema registry.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
