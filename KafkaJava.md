  

# Quickstart with OCI Java SDK for OSS

This quickstart shows how to produce messages to and consume messages from an [Oracle Streaming Service](https://docs.oracle.com/en-us/iaas/Content/Streaming/Concepts/streamingoverview.htm) using the [Kafka Java Client](https://docs.confluent.io/clients-kafka-java/current/overview.html). Please note, OSS is API compatible with Apache Kafka. Hence developers who are already familiar with Kafka need to make only few minimal changes to their Kafka client code, like config values like endpoint for Kafka brokers!

## Prerequisites

1. You need have [OCI account subscription or free account](https://www.oracle.com/cloud/free/). 
2. Follow  [these steps](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md)  to create Streampool and Stream in OCI. If you do already have stream created, refer step 4 [here](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md)  to capture information related to  `Kafka Connection Settings`. We need this Information for upcoming steps.
3. JDK 8 or above installed. Make sure *java* is in your PATH.
4. Maven 3.0 or installed. Make sure *mvn* is in your PATH. 
5. [Intellij Idea](https://www.jetbrains.com/idea/) (recommended) or any other integrated development environment (IDE).
6. Add the latest version of maven dependency or jar for [Kafka Java SDK](https://mvnrepository.com/artifact/org.apache.kafka/kafka-clients) to your *pom.xml* as shown below.
```Xml
	<dependency>
		<groupId>org.apache.kafka</groupId>
		<artifactId>kafka-clients</artifactId>
		<version>2.8.0</version>
	</dependency>
```
7. Assuming *wd* as your working directory for your Java project of this example, your *pom.xml* will look similar to one shown below.
```Xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>oci.example</groupId>
    <artifactId>StreamsExampleWithKafkaApis</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>
		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-clients</artifactId>
			<version>2.8.0</version>
		</dependency>
    </dependencies>
</project>
```
8.  Authentication with the Kafka protocol uses auth-tokens and the SASL/PLAIN mechanism. Follow  [Working with Auth Tokens](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcredentials.htm#Working)  for auth-token generation. Since you have created the stream(aka Kafka Topic) and Streampool in OCI, you are already authorized to use this stream as per OCI IAM. Hence create auth-token for your user in OCI. These  `OCI user auth-tokens`  are visible only once at the time of creation. Hence please copy it and keep it somewhere safe, as we are going to need it later.

## Producing messages to OSS
1. Open your favorite editor, such as *Intellij Idea* from the directory *wd*. You should already have Kafka dependencies for Java as part of your *pom.xml* of your maven java project  (as per the *step 6, step 7 of Prerequisites* section).
2. Create new file named *Producer.java* in `wd` directory under the path `/src/main/java/kafka/sdk/oss/example/` and paste the following code in it. You also need to replace values of static variables in the code namely `bootstrapServers` to `streamOrKafkaTopicName`, as directed by code comments .  These variables are for Kafka connection settings. You should already have all the this info and topic name(stream name) from the step 2 of the Prerequisites section of this tutorial.
```Java
package kafka.sdk.oss.example;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class Producer {

    static String bootstrapServers = "<end point of the bootstrap servers>", //usually of the form cell-1.streaming.[region code].oci.oraclecloud.com:9092 ;
    static String tenancyName = "<YOUR_TENANCY_NAME>";
    static String username = "<YOUR_OCI_USERNAME>";
    static String streamPoolId = "<OCID_FOR_STREAMPOOL_OF_THE_STREAM>";
    static String authToken = "<YOUR_OCI_AUTH_TOKEN>"; // from step 8 of Prerequisites section
    static String streamOrKafkaTopicName = "<YOUR_STREAM_NAME>"; // from step 2 of Prerequisites section

    private static Properties getKafkaProperties() {
        Properties properties = new Properties();
        properties.put("bootstrap.servers", bootstrapServers);
        properties.put("security.protocol", "SASL_SSL");
        properties.put("sasl.mechanism", "PLAIN");
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        final String value = "org.apache.kafka.common.security.plain.PlainLoginModule required username=\""
                + tenancyName + "/"
                + username + "/"
                + streamPoolId + "\" "
                + "password=\""
                + authToken + "\";";
        properties.put("sasl.jaas.config", value);
        properties.put("retries", 3); // retries on transient errors and load balancing disconnection
        properties.put("max.request.size", 1024 * 1024); // limit request size to 1MB
        return properties;
    }

    public static void main(String args[]) {
        try {
            Properties properties = getKafkaProperties();
            KafkaProducer producer = new KafkaProducer<>(properties);

            for(int i=0;i<10;i++) {
                ProducerRecord<String, String> record = new ProducerRecord<>(streamOrKafkaTopicName, "messageKey" + i, "messageValue" + i);
                producer.send(record, (md, ex) -> {
                    if (ex != null) {
                        System.err.println("exception occurred in producer for review :" + record.value()
                                + ", exception is " + ex);
                        ex.printStackTrace();
                    } else {
                        System.err.println("Sent msg to " + md.partition() + " with offset " + md.offset() + " at " + md.timestamp());
                    }
                });
            }
            // producer.send() is async, to make sure all messages are sent we use producer.flush()
            producer.flush();
            producer.close();
        } catch (Exception e) {
            System.err.println("Error: exception " + e);
        }
    }
}
```
3.   Run the code on the terminal(from the same directory *wd*) follows 
```Shell
mvn clean install exec:java -Dexec.mainClass=kafka.sdk.oss.example.Producer
```
4. In the OCI Web Console, quickly go to your Stream Page and click on *Load Messages* button. You should see the messages we just produced as below.
![See Produced Messages in OCI Wb Console](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/StreamExampleLoadMessages.png?raw=true)

  
## Consuming messages from OSS
1. First produce messages to the stream you want to consumer message from unless you already have messages in the stream. You can produce message easily from *OCI Web Console* using simple *Produce Test Message* button as shown below
![Produce Test Message Button](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/ProduceButton.png?raw=true)
 
 You can produce multiple test messages by clicking *Produce* button back to back, as shown below
![Produce multiple test message by clicking Produce button](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/ActualProduceMessagePopUp.png?raw=true)

2. Open your favorite editor, such as *Intellij Idea* from the directory *wd*. You should already have oci-sdk dependencies for Java as part of your *pom.xml* of your maven java project  (as per the *step 6, step 7 of Prerequisites* section).

3. Create new file named *Consumer.java* in directory *wd* directory under the path `/src/main/java/kafka/sdk/oss/example/` and paste the following code in it. You also need to replace values of static variables in the code namely `bootstrapServers` to `streamOrKafkaTopicName`, as directed by code comments . These variables are for Kafka connection settings. You should already have all the this info and topic name(stream name) from the step 2 of the Prerequisites section of this tutorial.
```Java
package kafka.sdk.oss.example;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class Consumer {
    static String bootstrapServers = "<end point of the bootstrap servers>", //usually of the form cell-1.streaming.[region code].oci.oraclecloud.com:9092 ;
    static String tenancyName = "<YOUR_TENANCY_NAME>";
    static String username ="<YOUR_OCI_USERNAME>";
    static String streamPoolId = "<OCID_FOR_STREAMPOOL_OF_THE_STREAM>";
    static String authToken = "<YOUR_OCI_AUTH_TOKEN>"; // from step 8 of Prerequisites section
    static String streamOrKafkaTopicName = "<YOUR_STREAM_NAME>"; // from step 2 of Prerequisites section
    static String consumerGroupName = "<any arbitrary consumer group name>"; 

    private static Properties getKafkaProperties(){
        Properties props = new Properties();
        props.put("bootstrap.servers", bootstrapServers);
        props.put("group.id", consumerGroupName);
        props.put("enable.auto.commit", "false");
        props.put("session.timeout.ms", "30000");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("security.protocol", "SASL_SSL");
        props.put("sasl.mechanism", "PLAIN");
        props.put("auto.offset.reset", "earliest");
        final String value = "org.apache.kafka.common.security.plain.PlainLoginModule required username=\""
                + tenancyName + "/"
                + username + "/"
                + streamPoolId + "\" "
                + "password=\""
                + authToken + "\";";
        props.put("sasl.jaas.config", value);
        return props;
    }

    public static void main(String[] args) {
        final KafkaConsumer<Integer, String> consumer = new KafkaConsumer<>(getKafkaProperties());;
        consumer.subscribe(Collections.singletonList(streamOrKafkaTopicName));
        ConsumerRecords<Integer, String> records = consumer.poll(10000);

        System.out.println("size of records polled is "+ records.count());
        for (ConsumerRecord<Integer, String> record : records) {
            System.out.println("Received message: (" + record.key() + ", " + record.value() + ") at offset " + record.offset());
        }

        consumer.commitSync();
        consumer.close();
    }
}

```
4. Run the code on the terminal(from the same directory *wd*) follows 
```Shell
mvn clean install exec:java -Dexec.mainClass=kafka.sdk.oss.example.Consumer
```
5. You should see the messages similar to shown below. Note when we produce message from OCI Web Console(as described above in first step), the Key for each message is *Null*
```
$:/path/to/directory/wd>mvn clean install exec:java -Dexec.mainClass=kafka.sdk.oss.example.Consumer
[INFO related maven compiling and building the Java code]
size of records polled is 3
Received message: (messageKey0, message value) at offset 1284
Received message: (messageKey0, message value) at offset 1285
Received message: (null, message produced using oci console) at offset 1286

```

## Next Steps
Please refer to

 1. [Kafka Producer API Java Docs](https://kafka.apache.org/22/javadoc/org/apache/kafka/clients/producer/ProducerConfig.html) 
 2. [Kafka Consumer API Java Docs](https://kafka.apache.org/22/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)
 3. [Kafka API support for OSS](https://docs.oracle.com/en-us/iaas/Content/Streaming/Tasks/kafkacompatibility.htm#kafka_apis)
