# Kafka Connect

## Definitions
- __Kafka Connect__ is a framework for connecting Kafka with external systems such as databases, key-value stores, search indexes, and file systems.

- __Source Connector__: ingests entire databases and streams table updates to Kafka topics.

- __Sink Connector__: delivers data from Kafka topics into secondary indexes such as Elasticsearch, or batch systems such as Hadoop for offline analysis.

# Features
- __A common framework for Kafka connectors__: Kafka Connect standardizes integration of other data systems with Kafka, simplifying connector development, deployment, and management.

- __Distributed and scalable by default__: Kafka Connect builds on the existing group management protocol. More workers can be added to scale up a Kafka Connect cluster.

- __Streaming/batch integration__: leveraging Kafka's existing capabilities, Kafka Connect is an ideal solution for bridging streaming and batch data systems

- __REST interface__: submit and manage connectors to your Kafka Connect cluster via an easy to use REST API.

#
## Concepts

1. __Connectors__: Connectors in Kafka Connect __define where data should be copied to and from__. A connector instance is __a logical job__ that is responsible for managing the copying of data between Kafka and another system.
    - It is recommended to leverage existing connectors.
    - To write a new connector plugin follows the workflow below:

    ![](references/img/connector-model-simple.png)

2. __Tasks__: Each connector instance coordinates a set of tasks that actually copy the data. These tasks have no state stored within them. Task state is stored in Kafka in special topics ```config.storage.topic``` and ```status.storage.topic``` and managed by the associated connector.

3. __Task Rebalancing__: When a connector is first submitted to the cluster, the workers rebalance the full set of connectors in the cluster and their tasks so that each worker has approximately the same amount of work.
    - When a __worker fails__, tasks are rebalanced across the active workers
    - When a __task fails__, no rebalance is triggered as a task failure is considered an exceptional case. As such, __failed tasks are not automatically restarted__ by the framework and should be restarted via the REST API.

    ![](references/img/task-failover.png)

4. __Workers__: Connectors and tasks are logical units of work and must be scheduled to execute in a process. Kafka Connect calls these processes workers and has two types of workers: standalone and distributed.

    - __Standalone Workers__: a single process is responsible for executing all connectors and tasks. Limited functionality: scalability is limited to the single process and there is no fault tolerance beyond any monitoring you add to the single process.


    - __Distributed Workers__: Provides scalability and automatic fault tolerance for Kafka Connect. In distributed mode, you start many worker processes using the same ```group.id```. If you add a worker, shut down a worker, or a worker fails unexpectedly, the rest of the workers detect this and automatically coordinate to redistribute connectors and tasks across the updated set of available workers. 

5. __Converters__: Particular data formats when writing to or reading from Kafka. Tasks use converters to change the format of data from bytes (Kafka) to a Connect internal data format and vice versa.
    - By default, Confluent Platform provides the following converters:
        - AvroConverter: use with Schema Registry
        - ProtobufConverter: use with Schema Registry
        - JsonSchemaConverter: use with Schema Registry
        - JsonConverter: with out Schame Registry
        - StringConverter: simple format
        - ByteArrayConverter: no conversion.
        
    ![](references/img/converter-basics.png)

6. __Transforms__: A transform is a simple function that accepts one record as an input and outputs a modified record.  

    - You can implement the Transformation interface with your own custom logic, package them as a Kafka Connect plugin, and use them with any connectors.

    - When transforms are used with a source connector, Kafka Connect passes each source record produced by the connector through the first transformation, which makes its modifications and outputs a new source record. This updated source record is then passed to the next transform in the chain, which generates a new modified source record. This continues for the remaining transforms. The final updated source record is converted to the binary form and written to Kafka.

7. __Dead Letter Queue__: An invalid record may occur for a number of reasons. One example is when a record arrives at the __sink connector serialized in JSON format__, but the sink connector configuration is __expecting Avro format__. When an invalid record cannot be processed by a sink connector, the error is handled based on the connector configuration property ```errors.tolerance```
    - Dead letter queues are __only applicable for sink connectors__.

    - ```"errors.tolerance": "none"``` - an error or invalid record causes the connector task to immediately fail and the connector goes into a failed state. To resolve this issue, you would need to review the Kafka Connect Worker log to find out what caused the failure, correct it, and restart the connector. (not recommend)
    - ```"errors.tolerance": "none"``` - All errors or invalid records are __ignored and processing continues__. __No errors are written to the Connect Worker log.__

    - An error-handling feature is available that will route all __invalid records to a special topic__ and report the error. __This topic contains a dead letter queue__ of records that could not be processed by the sink connector.
    
    
    ```
    {
        "name": "gcs-sink-01",
        "config": {
            "connector.class": "io.confluent.connect.gcs.GcsSinkConnector",
            "tasks.max": "1",
            "topics": "gcs_topic",
            "gcs.bucket.name": "<my-gcs-bucket>",
            "gcs.part.size": "5242880",
            "flush.size": "3",
            "storage.class": "io.confluent.connect.gcs.storage.GcsStorage",
            "format.class": "io.confluent.connect.gcs.format.avro.AvroFormat",
            "partitioner.class": "io.confluent.connect.storage.partitioner.DefaultPartitioner",
            "value.converter": "io.confluent.connect.avro.AvroConverter",
            "value.converter.schema.registry.url": "http://localhost:8081",
            "schema.compatibility": "NONE",
            "confluent.topic.bootstrap.servers": "localhost:9092",
            "confluent.topic.replication.factor": "1",
            "errors.tolerance": "all",                              // ignore invalid records
            "errors.deadletterqueue.topic.name": "dlq-gcs-sink-01", // create dead letter topic
            "errors.deadletterqueue.context.headers.enable":true    // show the fail reason in record headers
        }
    }
    ```
 