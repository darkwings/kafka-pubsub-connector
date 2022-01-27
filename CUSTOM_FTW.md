# Customization NTT

This connector is based on the original GCP Cloud Connector, with some modifications.

First, we removed the PubSub LITE classes.

## Connectors configuration

Some new properties were added in order to satisfy some use cases not supported by the original connector.

### SOURCE Connector

* ```sink.dlq.topic```: the DLQ topic to which redirect the message that generates an error at application level (e.g. a failure in the flush). This behaviour needs improvement.

### SINK Connector

* ```cps.as.plain.string```: allows to forward the messages read from Pubsub subscription to Kafka as plain string instead of bytes. If not declared, or if set to ```false```, the behaviour of the connector is the original one.


## How to test

### Install Kafka (Zookeeper, Broker and Connect)
Start an instance of Kafka broker (with Zookeeper) and Connect worker. 
You can install either Kafka Open Source or the Confluent Platform.
Please follow the instruction for the version of your choice. Regarding Kafka Connect, you can start 
either in *standalone* mode or *distributed* mode. Keep in mind that in production environment is recommended
to use the distributed mode, so we assume that you're going to use this mode. The distributed mode is also the 
default mode in which Kafka Connect starts if you use Confluent Platform.

Whichever version of Kafka Connect you choose, enable Rest API in the configuration file with ```listeners=HTTP://localhost:8083```.

#### Kafka OS

To run Kafka locally (or on bare metal), download and unzip the distribution from Kafka website. Then, execute the
following commands from the directory in which you unzipped the package.

     bin/zookeeper-server-start.sh config/zookeeper.properties
     bin/kafka-server-start.sh config/server.properties
     bin/connect-distributed.sh config/connect-distributed.properties

Keep in mind updating ```connect-distributed.properties``` adding ```listeners``` and ```plugin.path``` properties (see below).

### Connector

Prepare the jar with the command

     mvn clean package

You will find the target jar in the target directory. Copy the jar in the proper directory

- if you installed Confluent Platform copy the file in ```CONFLUENT_HOME/share/java```. 
- if you installed Kafka OS, place the jar in a directory of your choice. You have to specify the directory in the configuration file ```config/connect-distributed.properties```, in the ```plugin.path``` property.

### Configure the connector

#### Sink Connector

Execute the following command in a shell

     curl -X POST http://localhost:8083/connectors -H 'Content-Type:application/json' -H 'Accept:application/json' \
        -d '{
            "name": "PubSubSinkConnectorJSON",
            "config": {
                "connector.class": "com.ftw.pubsub.kafka.sink.CloudPubSubSinkConnector",
                "tasks.max": "4",
                "topics": "topic-vf-json",
                "errors.tolerance":"all",
                "errors.deadletterqueue.topic.name":"topic-vf-json-dlq",
                "cps.topic":"test-vf-json",
                "cps.project":"big-query-test-332316",
                "gcp.credentials.file.path":"/path/to/service-account.json",
                "key.converter":"org.apache.kafka.connect.storage.StringConverter",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable":"false"
            }
        }'

#### Source Connector

Execute the following command in a shell. You can use a Pubsub message attribute as key 
setting the property ```kafka.key.attribute```. 

     curl -X POST http://localhost:8083/connectors -H 'Content-Type:application/json' -H 'Accept:application/json' \
        -d '{
            "name": "PubSubSourceConnectorJSON",
            "config": {
                "connector.class": "com.ftw.pubsub.kafka.source.CloudPubSubSourceConnector",
                "tasks.max": "1",
                "kafka.topic":"json-from-pubsub",
                "kafka.key.attribute":"key",
                "errors.tolerance":"all",
                "errors.deadletterqueue.topic.name":"json-from-pubsub-dlq",
                "cps.subscription":"cps-vf-json-sub",
                "cps.project":"big-query-test-332316",
                "cps.as.plain.string":"true",
                "gcp.credentials.file.path":"/path/to/service-account.json",
                "key.converter":"org.apache.kafka.connect.storage.StringConverter",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable":"false"
            }
        }'