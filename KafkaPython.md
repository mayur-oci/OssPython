

# Quickstart with Kafka Python Clients for OSS

This quickstart shows how to produce messages to and consume messages from an [**Oracle Streaming Service**](https://docs.oracle.com/en-us/iaas/Content/Streaming/Concepts/streamingoverview.htm) using the [Kafka Python Client](https://docs.confluent.io/clients-confluent-kafka-python/current/overview.html). Please note, OSS is API compatible with Apache Kafka. Hence developers who are already familiar with Kafka need to make only few minimal changes to their Kafka client code, like config values like endpoint for Kafka brokers!

## Prerequisites

1. You need have [OCI account subscription or free account](https://www.oracle.com/cloud/free/). 
2. Follow  [these steps](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md)  to create Streampool and Stream in OCI. If you do already have stream created, refer step 4  [here](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md)  to capture information related to  `Kafka Connection Settings`. We need this Information for upcoming steps.
3. Python 3.6 or later, with PIP installed and updated.
4. Visual Studio Code(recommended) or any other integrated development environment (IDE).
5. Install Confluent-Kafka packages for Python as follows. 
```
      pip install confluent-kafka
```
You can install it globally, or within a [virtualenv](https://docs.python.org/3/library/venv.html). 
*librdkafka* pakcage is used by above *confluent-kafka* package and it is embedded in wheels for latest release of *confluent-kafka*. For more detail refer [here](https://github.com/confluentinc/confluent-kafka-python/blob/master/README.md#prerequisites).

6. You need to install the CA root certificates on your host(where you are going developing and running this quickstart). The client will use CA certificates to verify the broker's certificate. For all platforms exept Windows, please follow [here](https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/python.html#configure-ssl-trust-store). For [Windows](https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/csharp.html#prerequisites), please download `cacert.pem` file distributed with curl ([download cacert.pm](https://curl.haxx.se/ca/cacert.pem)). 
7.  Authentication with the Kafka protocol uses auth-tokens and the SASL/PLAIN mechanism. Follow  [Working with Auth Tokens](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcredentials.htm#Working)  for auth-token generation. Since you have created the stream(aka Kafka Topic) and Streampool in OCI, you are already authorized to use this stream as per OCI IAM. Hence create auth-token for your user in OCI. These  `OCI user auth-tokens`  are visible only once at the time of creation. Hence please copy it and keep it somewhere safe, as we are going to need it later.

## Producing messages to OSS
1. Open your favorite editor, such as [Visual Studio Code](https://code.visualstudio.com) from the directory *wd*. You should already have oci-sdk packages for Python installed for your current python environment (as per the *step 5 of Prerequisites* section).
2. Create new file named *Producer.py* in this directory and paste the following code in it.
```Python

import certifi  
from confluent_kafka import Producer, KafkaError  
  
if __name__ == '__main__':  
  
    # Read arguments and configurations and initialize  
  topic = "StreamExample"  
  conf = {  
        'bootstrap.servers': 'cell-1.streaming.ap-mumbai-1.oci.oraclecloud.com:9092', # replace  
  'security.protocol': 'SASL_SSL',  
  'ssl.ca.location': certifi.where(), # replace '/usr/local/etc/openssl@1.1/cert.pem'  
  'sasl.mechanism': 'PLAIN',  
  'sasl.username': 'intrandallbarnes/mayur.raleraskar@oracle.com/ocid1.streampool.oc1.ap-mumbai-1.amaaaaaauwpiejqaf6bqruy7ljuhfrtppqf5jyy22g5yu4cqfzg2ik5uqu6q',  
  'sasl.password': '2m{s4WTCXysp:o]tGx4K', # replace  
 # 'client.id': 'python-example-producer'  }  
  
    # Create Producer instance  
  producer = Producer(**conf)  
    delivered_records = 0  
  
  # Optional per-message on_delivery handler (triggered by poll() or flush())  
 # when a message has been successfully delivered or permanently failed delivery after retries.  def acked(err, msg):  
        global delivered_records  
        """Delivery report handler called on  
 successful or failed delivery of message """  if err is not None:  
            print("Failed to deliver message: {}".format(err))  
        else:  
            delivered_records += 1  
  print("Produced record to topic {} partition [{}] @ offset {}"  
  .format(msg.topic(), msg.partition(), msg.offset()))  
  
  
    for n in range(10):  
        record_key = "messageKey" + str(n)  
        record_value = "messageValue" + str(n)  
        print("Producing record: {}\t{}".format(record_key, record_value))  
        producer.produce(topic, key=record_key, value=record_value, on_delivery=acked)  
        # p.poll() serves delivery reports (on_delivery) from previous produce() calls.  
  producer.poll(0)  
  
    producer.flush()  
  
    print("{} messages were produced to topic {}!".format(delivered_records, topic))
```
3.   Run the code on the terminal(from the same directory *wd*) follows 
```
python Producer.py
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

 1. [Github for Confluent Python SDK](https://github.com/confluentinc/confluent-kafka-python)
