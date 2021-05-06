

# Quickstart with OCI Java SDK for OSS

This quickstart shows how to produce messages to and consume messages from an [**Oracle Streaming Service**](https://docs.oracle.com/en-us/iaas/Content/Streaming/Concepts/streamingoverview.htm) using the [OCI Java SDK](https://github.com/oracle/oci-java-sdk).

## Prerequisites

1. You need have OCI account subscription or free account. typical links @jb
2. Follow [these steps](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md) to create Streampool and Stream in OCI. If you do  already have stream created, refer step 3 [here](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md) to capture/record message endpoint and OCID of the stream. We need this Information for upcoming steps.
3. JDK 8 or above installed. Make sure Java is in your PATH.
4. Maven 3.0 or installed. Make sure Maven is in your PATH. 
5. Intellij(recommended) or any other integrated development environment (IDE).
6. Add the latest version of maven dependency or jar for [OCI Java SDK for IAM](https://search.maven.org/artifact/com.oracle.oci.sdk/oci-java-sdk-common/) to your *pom.xml* as shown below.
```Xml
	<dependency>
	  <groupId>com.oracle.oci.sdk</groupId>
	  <artifactId>oci-java-sdk-common</artifactId>
	  <version>LATEST</version>
	</dependency>
```
7. Add the latest version of maven dependency or jar for [OCI Java SDK for OSS](https://search.maven.org/artifact/com.oracle.oci.sdk/oci-java-sdk-streaming) to your *pom.xml* as shown below..
```Xml
	<dependency>
	  <groupId>com.oracle.oci.sdk</groupId>
	  <artifactId>oci-java-sdk-streaming</artifactId>
	  <version>LATEST</version> 
	</dependency>
```
8. Assuming *wd* as your working directory for your Java project of this example, your *pom.xml* will look similar to one shown below
```Xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>oci.example</groupId>
    <artifactId>StreamsJava</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.oracle.oci.sdk</groupId>
            <artifactId>oci-java-sdk-common</artifactId>
            <version>1.33.2</version>
        </dependency>
        <dependency>
            <groupId>com.oracle.oci.sdk</groupId>
            <artifactId>oci-java-sdk-streaming</artifactId>
            <version>1.33.2</version>
        </dependency>
    </dependencies>
</project>
```
10. Make sure you have [SDK and CLI Configuration File](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdkconfig.htm#SDK_and_CLI_Configuration_File) setup. For production, you should use [Instance Principle Authentication](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm).

## Producing messages to OSS
1. Open your favorite editor, such as [Visual Studio Code](https://code.visualstudio.com) from the directory *wd*. You should already have oci-sdk dependencies for Java as part of your *pom.xml* of your maven java project  (as per the *step 6, step 7 of Prerequisites* section).
2. Create new file named *Producer.java* in directory *wd* with following code after you replace values of variables configurationFilePath, profile ,ociStreamOcid and ociMessageEndpoint in the follwing code snippet with values applicable for your tenancy. 
```Java
package oci.sdk.oss.example;

import com.oracle.bmc.ConfigFileReader;
import com.oracle.bmc.auth.AuthenticationDetailsProvider;
import com.oracle.bmc.auth.ConfigFileAuthenticationDetailsProvider;
import com.oracle.bmc.streaming.StreamClient;
import com.oracle.bmc.streaming.model.PutMessagesDetails;
import com.oracle.bmc.streaming.model.PutMessagesDetailsEntry;
import com.oracle.bmc.streaming.model.PutMessagesResultEntry;
import com.oracle.bmc.streaming.requests.PutMessagesRequest;
import com.oracle.bmc.streaming.responses.PutMessagesResponse;
import org.apache.commons.lang3.StringUtils;

import java.util.ArrayList;
import java.util.List;

import static java.nio.charset.StandardCharsets.UTF_8;

public class Producer {
    public static void main(String[] args) throws Exception {
        final String configurationFilePath = "~/.oci/config";
        final String profile = "DEFAULT";
        final String ociStreamOcid = "ocid1.stream.oc1.ap-mumbai-1." +
                "amaaaaaauwpiejqaxcfc2ht67wwohfg7mxcstfkh2kp3hweeenb3zxtr5khq";
        final String ociMessageEndpoint = "https://cell-1.streaming.ap-mumbai-1.oci.oraclecloud.com";


        final ConfigFileReader.ConfigFile configFile = ConfigFileReader.parseDefault();
        final AuthenticationDetailsProvider provider =
                new ConfigFileAuthenticationDetailsProvider(configFile);

        // Streams are assigned a specific endpoint url based on where they are provisioned.
        // Create a stream client using the provided message endpoint.
        StreamClient streamClient = StreamClient.builder().endpoint(ociMessageEndpoint).build(provider);

        // publish some messages to the stream
        publishExampleMessages(streamClient, ociStreamOcid);

    }

    private static void publishExampleMessages(StreamClient streamClient, String streamId) {
        // build up a putRequest and publish some messages to the stream
        List<PutMessagesDetailsEntry> messages = new ArrayList<>();
        for (int i = 0; i < 50; i++) {
            messages.add(
                    PutMessagesDetailsEntry.builder()
                            .key(String.format("messageKey%s", i).getBytes(UTF_8))
                            .value(String.format("messageValue%s", i).getBytes(UTF_8))
                            .build());
        }

        System.out.println(
                String.format("Publishing %s messages to stream %s.", messages.size(), streamId));
        PutMessagesDetails messagesDetails =
                PutMessagesDetails.builder().messages(messages).build();

        PutMessagesRequest putRequest =
                PutMessagesRequest.builder()
                        .streamId(streamId)
                        .putMessagesDetails(messagesDetails)
                        .build();

        PutMessagesResponse putResponse = streamClient.putMessages(putRequest);

        // the putResponse can contain some useful metadata for handling failures
        for (PutMessagesResultEntry entry : putResponse.getPutMessagesResult().getEntries()) {
            if (StringUtils.isNotBlank(entry.getError())) {
                System.out.println(
                        String.format("Error(%s): %s", entry.getError(), entry.getErrorMessage()));
            } else {
                System.out.println(
                        String.format(
                                "Published message to partition %s, offset %s.",
                                entry.getPartition(),
                                entry.getOffset()));
            }
        }
    }


}


```
3.   Run the code on the terminal(from the same directory *wd*) follows 
```Shell
mvn install exec:java -Dexec.mainClass=oci.sdk.oss.example.Producer
```
4. In the OCI Web Console, quickly go to your Stream Page and click on *Load Messages* button. You should see the messages we just produced as below.
![See Produced Messages in OCI Wb Console](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/StreamExampleLoadMessages.png?raw=true)

  
## Consuming messages from OSS
1. First produce messages to the stream you want to consumer message from unless you already have messages in the stream. You can produce message easily from *OCI Web Console* using simple *Produce Test Message* button as shown below
![Produce Test Message Button](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/ProduceButton.png?raw=true)
 
 You can produce multiple test messages by clicking *Produce* button back to back, as shown below
![Produce multiple test message by clicking Produce button](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/ActualProduceMessagePopUp.png?raw=true)

2. Open your favorite editor, such as [Visual Studio Code](https://code.visualstudio.com) from the directory *wd*. You should already have oci-sdk dependencies for Java as part of your *pom.xml* of your maven java project  (as per the *step 6, step 7 of Prerequisites* section).

3. Create new file named *Consumer.java* in directory *wd* with following code after you replace values of variables configurationFilePath, profile ,ociStreamOcid and ociMessageEndpoint in the follwing code snippet with values applicable for your tenancy. 
```Java
package oci.sdk.oss.example;

import com.google.common.util.concurrent.Uninterruptibles;
import com.oracle.bmc.ConfigFileReader;
import com.oracle.bmc.auth.AuthenticationDetailsProvider;
import com.oracle.bmc.auth.ConfigFileAuthenticationDetailsProvider;
import com.oracle.bmc.streaming.StreamClient;
import com.oracle.bmc.streaming.model.CreateGroupCursorDetails;
import com.oracle.bmc.streaming.model.Message;
import com.oracle.bmc.streaming.requests.CreateGroupCursorRequest;
import com.oracle.bmc.streaming.requests.GetMessagesRequest;
import com.oracle.bmc.streaming.responses.CreateGroupCursorResponse;
import com.oracle.bmc.streaming.responses.GetMessagesResponse;

import java.util.concurrent.TimeUnit;

import static java.nio.charset.StandardCharsets.UTF_8;


public class Consumer {
    public static void main(String[] args) throws Exception {
        final String configurationFilePath = "~/.oci/config";
        final String profile = "DEFAULT";
        final String ociStreamOcid = "ocid1.stream.oc1.ap-mumbai-1." +
                "amaaaaaauwpiejqaxcfc2ht67wwohfg7mxcstfkh2kp3hweeenb3zxtr5khq";
        final String ociMessageEndpoint = "https://cell-1.streaming.ap-mumbai-1.oci.oraclecloud.com";

        final ConfigFileReader.ConfigFile configFile = ConfigFileReader.parseDefault();
        final AuthenticationDetailsProvider provider =
                new ConfigFileAuthenticationDetailsProvider(configFile);

        // Streams are assigned a specific endpoint url based on where they are provisioned.
        // Create a stream client using the provided message endpoint.
        StreamClient streamClient = StreamClient.builder().endpoint(ociMessageEndpoint).build(provider);

        // A cursor can be created as part of a consumer group.
        // Committed offsets are managed for the group, and partitions
        // are dynamically balanced amongst consumers in the group.
        System.out.println("Starting a simple message loop with a group cursor");
        String groupCursor =
                getCursorByGroup(streamClient, ociStreamOcid, "exampleGroup", "exampleInstance-1");
        simpleMessageLoop(streamClient, ociStreamOcid, groupCursor);

    }

    private static void simpleMessageLoop(
            StreamClient streamClient, String streamId, String initialCursor) {
        String cursor = initialCursor;
        for (int i = 0; i < 10; i++) {

            GetMessagesRequest getRequest =
                    GetMessagesRequest.builder()
                            .streamId(streamId)
                            .cursor(cursor)
                            .limit(25)
                            .build();

            GetMessagesResponse getResponse = streamClient.getMessages(getRequest);

            // process the messages
            System.out.println(String.format("Read %s messages.", getResponse.getItems().size()));
            for (Message message : ((GetMessagesResponse) getResponse).getItems()) {
                System.out.println(
                        String.format(
                                "%s: %s",
                                message.getKey() == null ? "Null" :new String(message.getKey(), UTF_8),
                                new String(message.getValue(), UTF_8)));
            }

            // getMessages is a throttled method; clients should retrieve sufficiently large message
            // batches, as to avoid too many http requests.
            Uninterruptibles.sleepUninterruptibly(1, TimeUnit.SECONDS);

            // use the next-cursor for iteration
            cursor = getResponse.getOpcNextCursor();
        }
    }

    private static String getCursorByGroup(
            StreamClient streamClient, String streamId, String groupName, String instanceName) {
        System.out.println(
                String.format(
                        "Creating a cursor for group %s, instance %s.", groupName, instanceName));

        CreateGroupCursorDetails cursorDetails =
                CreateGroupCursorDetails.builder()
                        .groupName(groupName)
                        .instanceName(instanceName)
                        .type(CreateGroupCursorDetails.Type.TrimHorizon)
                        .commitOnGet(true)
                        .build();

        CreateGroupCursorRequest createCursorRequest =
                CreateGroupCursorRequest.builder()
                        .streamId(streamId)
                        .createGroupCursorDetails(cursorDetails)
                        .build();

        CreateGroupCursorResponse groupCursorResponse =
                streamClient.createGroupCursor(createCursorRequest);
        return groupCursorResponse.getCursor().getValue();
    }

}

```
4. Run the code on the terminal(from the same directory *wd*) follows 
```Shell
mvn install exec:java -Dexec.mainClass=oci.sdk.oss.example.Consumer
```
5. You should see the messages similar to shown below. Note when we produce message from OCI Web Console(as described above in first step), the Key for each message is *Null*
```
$:/path/to/directory/wd>mvn install exec:java -Dexec.mainClass=oci.sdk.oss.example.Consumer
 [INFO related maven compiling and building the Java code]
Starting a simple message loop with a group cursor
Creating a cursor for group exampleGroup, instance exampleInstance-1.
Read 25 messages.
Null: Example Test Message 0
Null: Example Test Message 0
 Read 2 messages
Null: Example Test Message 0
Null: Example Test Message 0
 Read 1 messages
Null: Example Test Message 0
 Read 10 messages
key 0: value 0
key 1: value 1

```

## Next Steps
Please refer to

 1. [Github for OCI Java SDK](https://github.com/oracle/oci-java-sdk)
 2. [Streaming Examples with Admin and Client APIs from OCI](https://github.com/oracle/oci-java-sdk/blob/master/bmc-examples/src/main/java/StreamsExample.java)
