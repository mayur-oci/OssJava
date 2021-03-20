

# Quickstart with OCI Java SDK for OSS

This quickstart shows how to produce messages to and consume messages from an [**Oracle Streaming Service**](https://docs.oracle.com/en-us/iaas/Content/Streaming/Concepts/streamingoverview.htm) using the [OCI Java SDK](https://github.com/oracle/oci-java-sdk).

## Prerequisites

1. You need have OCI account subscription or free account. typical links @jb
2. Follow [these steps](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md) to create Streampool and Stream in OCI. If you do  already have stream created, refer step 3 [here](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md) to capture/record message endpoint and OCID of the stream. We need this Information for upcoming steps.
3. Java JDK 8 or above
4. Visual Studio Code(recommended) or any other integrated development environment (IDE).
5. Download the latest dependency or jar for [OCI Java SDK for IAM](https://search.maven.org/artifact/com.oracle.oci.sdk/oci-java-sdk-common/) and keep it in your classpath for your Java code. 
If you are using maven, add the following dependency to your pom. Get the latest version from maven [here](https://search.maven.org/artifact/com.oracle.oci.sdk/oci-java-sdk-streaming).
```Xml
	<dependency>
	  <groupId>com.oracle.oci.sdk</groupId>
	  <artifactId>oci-java-sdk-common</artifactId>
	  <version>1.33.2</version>
	</dependency>
```
6. Download the latest dependency or jar for [OCI Java SDK for OSS](https://search.maven.org/artifact/com.oracle.oci.sdk/oci-java-sdk-streaming) and keep it in your classpath for your Java code. 
If you are using maven, add the following dependency to your pom. Get the latest version from maven [here](https://search.maven.org/artifact/com.oracle.oci.sdk/oci-java-sdk-streaming).
```Xml
	<dependency>
	  <groupId>com.oracle.oci.sdk</groupId>
	  <artifactId>oci-java-sdk-streaming</artifactId>
	  <version>LATEST</version> 
	</dependency>
```
7. Make sure you have [SDK and CLI Configuration File](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdkconfig.htm#SDK_and_CLI_Configuration_File) setup. For production, you should use [Instance Principle Authentication](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm).

## Producing messages to OSS
1. Open your favorite editor, such as [Visual Studio Code](https://code.visualstudio.com) from the directory *wd*. You should already have oci-sdk packages for Python installed for your current python environment (as per the *step 5 of Prerequisites* section).
2. Create new file named *Producer.py* in this directory and paste the following code in it.
```Java


```
3.   Run the code on the terminal(from the same directory *wd*) follows 
```
java Producer.java
```
4. In the OCI Web Console, quickly go to your Stream Page and click on *Load Messages* button. You should see the messages we just produced as below.
![See Produced Messages in OCI Wb Console](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/StreamExampleLoadMessages.png?raw=true)

  
## Consuming messages from OSS
1. First produce messages to the stream you want to consumer message from unless you already have messages in the stream. You can produce message easily from *OCI Web Console* using simple *Produce Test Message* button as shown below
![Produce Test Message Button](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/ProduceButton.png?raw=true)
 
 You can produce multiple test messages by clicking *Produce* button back to back, as shown below
![Produce multiple test message by clicking Produce button](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/ActualProduceMessagePopUp.png?raw=true)
2. Open your favorite editor, such as [Visual Studio Code](https://code.visualstudio.com) from the directory *wd*. You should already have oci-sdk packages for Python installed for your current python environment as per the *step 5 of Prerequisites* section).
3. Create new file named *Consumer.py* in this directory and paste the following code in it.
```Python
import oci
import time

from base64 import b64decode

ociMessageEndpoint = "https://cell-1.streaming.ap-mumbai-1.oci.oraclecloud.com"
ociStreamOcid = "ocid1.stream.oc1.ap-mumbai-1.amaaaaaauwpiejqaxcfc2ht67wwohfg7mxcstfkh2kp3hweeenb3zxtr5khq"
ociConfigFilePath = "~/.oci/config"
ociProfileName = "DEFAULT"
compartment = ""


def get_cursor_by_group(sc, sid, group_name, instance_name):
    print(" Creating a cursor for group {}, instance {}".format(group_name, instance_name))
    cursor_details = oci.streaming.models.CreateGroupCursorDetails(group_name=group_name, instance_name=instance_name,
                                                                   type=oci.streaming.models.
                                                                   CreateGroupCursorDetails.TYPE_TRIM_HORIZON,
                                                                   commit_on_get=True)
    response = sc.create_group_cursor(sid, cursor_details)
    return response.data.value

def simple_message_loop(client, stream_id, initial_cursor):
    cursor = initial_cursor
    while True:
        get_response = client.get_messages(stream_id, cursor, limit=10)
        # No messages to process. return.
        if not get_response.data:
            return

        # Process the messages
        print(" Read {} messages".format(len(get_response.data)))
        for message in get_response.data:
            if message.key is None:
                key = "Null"
            else:
                key = b64decode(message.key.encode()).decode()
            print("{}: {}".format(key,
                                  b64decode(message.value.encode()).decode()))

        # get_messages is a throttled method; clients should retrieve sufficiently large message
        # batches, as to avoid too many http requests.
        time.sleep(1)
        # use the next-cursor for iteration
        cursor = get_response.headers["opc-next-cursor"]


config = oci.config.from_file(ociConfigFilePath, ociProfileName)
stream_client = oci.streaming.StreamClient(config, service_endpoint=ociMessageEndpoint)

# A cursor can be created as part of a consumer group.
# Committed offsets are managed for the group, and partitions
# are dynamically balanced amongst consumers in the group.
group_cursor = get_cursor_by_group(stream_client, ociStreamOcid, "example-group", "example-instance-1")
simple_message_loop(stream_client, ociStreamOcid, group_cursor)

```
4. Run the code on the terminal(from the same directory *wd*) follows 
```
python Consumer.py
```
5. You should see the messages as shown below. Note when we produce message from OCI Web Console(as described above in first step), the Key for each message is *Null*
```
$:/path/to/directory/wd>python Consumer.py 
 Creating a cursor for group example-group, instance example-instance-1
 Read 2 messages
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
