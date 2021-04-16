

# Quickstart with Kafka Python Clients for OSS

This quickstart shows how to produce messages to and consume messages from an [**Oracle Streaming Service**](https://docs.oracle.com/en-us/iaas/Content/Streaming/Concepts/streamingoverview.htm) using the [Kafka Python Client](https://docs.confluent.io/clients-confluent-kafka-python/current/overview.html). Please note, OSS is API compatible with Apache Kafka. Hence developers who are already familiar with Kafka need to make only few minimal changes to their Kafka client code, like config values like endpoint for Kafka brokers!

## Prerequisites

1. You need have [OCI account subscription or free account](https://www.oracle.com/cloud/free/). 
2. Follow  [these steps](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md)  to create Streampool and Stream in OCI. If you do already have stream created, refer step 4 [here](https://github.com/mayur-oci/OssJs/blob/main/JavaScript/CreateStream.md)  to capture information related to  `Kafka Connection Settings`. We need this Information for upcoming steps.
3. Python 3.6 or later, with PIP installed and updated.
4. Visual Studio Code(recommended) or any other integrated development environment (IDE).
5. Install Confluent-Kafka packages for Python as follows. 
```
      pip install confluent-kafka
```
You can install it globally, or within a [virtualenv](https://docs.python.org/3/library/venv.html). 
*librdkafka* pakcage is used by above *confluent-kafka* package and it is embedded in wheels for latest release of *confluent-kafka*. For more detail refer [here](https://github.com/confluentinc/confluent-kafka-python/blob/master/README.md#prerequisites).

6. You need to install the CA root certificates on your host(where you are going developing and running this quickstart). The client will use CA certificates to verify the broker's certificate. For all platforms exept Windows, please follow [here](https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/python.html#configure-ssl-trust-store). For [Windows](https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/csharp.html#prerequisites), please download `cacert.pem` file distributed with curl ([download cacert.pm](https://curl.haxx.se/ca/cacert.pem)). 

8.  Authentication with the Kafka protocol uses auth-tokens and the SASL/PLAIN mechanism. Follow  [Working with Auth Tokens](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcredentials.htm#Working)  for auth-token generation. Since you have created the stream(aka Kafka Topic) and Streampool in OCI, you are already authorized to use this stream as per OCI IAM. Hence create auth-token for your user in OCI. These  `OCI user auth-tokens`  are visible only once at the time of creation. Hence please copy it and keep it somewhere safe, as we are going to need it later.

## Producing messages to OSS
1. Open your favorite editor, such as [Visual Studio Code](https://code.visualstudio.com) from the directory *wd*. You should already have oci-sdk packages for Python installed for your current python environment (as per the *step 5 of Prerequisites* section).
2. Create new file named *Producer.py* in this directory and paste the following code in it. You also need to replace after you replace values of config variables in the map `conf` and the name of topic is the name of stream you created. You should already have all the Kafka config info and topic name(stream name) from the step 2 of the Prerequisites section of this tutorial.
```Python
 
from confluent_kafka import Producer, KafkaError  
  
if __name__ == '__main__':  
  
  topic = "[YOUR_STREAM_NAME]"  
  conf = {  
    'bootstrap.servers': "[end point of the bootstrap servers]", #usually of the form cell-1.streaming.[region code].oci.oraclecloud.com:9092  
    'security.protocol': 'SASL_SSL',  
  
    'ssl.ca.location': '/path/on/your/host/to/your/cert.pem/'  # from step 6 of Prerequisites section
     # optionally you can do 1. pip install certifi and 2. import certifi
     # ssl.ca.location: certifi.where()
  
    'sasl.mechanism': 'PLAIN',  
    'sasl.username': '[TENANCY_NAME]/[YOUR_OCI_USERNAME]/[OCID_FOR_STREAMPOOL]',  # from step 2 of Prerequisites section
    'sasl.password': '[YOUR_OCI_AUTH_TOKEN]',  # from step 7 of Prerequisites section
   }  
  
   # Create Producer instance  
   producer = Producer(**conf)  
   delivered_records = 0  
  
  # Optional per-message on_delivery handler (triggered by poll() or flush())  
  # when a message has been successfully delivered or permanently failed delivery after retries.  
  def acked(err, msg):  
        global delivered_records  
        """Delivery report handler called on  
            successful or failed delivery of message """  
        if err is not None:  
            print("Failed to deliver message: {}".format(err))  
        else:  
            delivered_records += 1  
            print("Produced record to topic {} partition [{}] @ offset {}".format(msg.topic(), msg.partition(), msg.offset()))  


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
2. Open your favorite editor, such as [Visual Studio Code](https://code.visualstudio.com) from the directory *wd*. You should already have *confluent-kafka* packages for Python installed for your current python environment as per the *step 5 of Prerequisites* section).

3. Create new file named *Consumer.py* in this directory and paste the following code in it. You also need to replace after you replace values of config variables in the map `conf` and the name of topic is the name of stream you created. You should already have all the Kafka config info and topic name(stream name) from the step 2 of the Prerequisites section of this tutorial.
```Python

from confluent_kafka import Consumer


if __name__ == '__main__':

  topic = "[YOUR_STREAM_NAME]"  
  conf = {  
    'bootstrap.servers': "[end point of the bootstrap servers]", #usually of the form cell-1.streaming.[region code].oci.oraclecloud.com:9092  
    'security.protocol': 'SASL_SSL',  
  
    'ssl.ca.location': '/path/on/your/host/to/your/cert.pem/'  # from step 6 of Prerequisites section
     # optionally you can do 1. pip install certifi and 2. import certifi
     # ssl.ca.location: certifi.where()
  
    'sasl.mechanism': 'PLAIN',  
    'sasl.username': '[TENANCY_NAME]/[YOUR_OCI_USERNAME]/[OCID_FOR_STREAMPOOL]',  # from step 2 of Prerequisites section
    'sasl.password': '[YOUR_OCI_AUTH_TOKEN]',  # from step 7 of Prerequisites section
    'group.id': 'python-example-group',
    'client.id': 'python-example-group-client',
    'default.topic.config': {'auto.offset.reset': 'smallest'}
    }

    # Create Consumer instance
    consumer = Consumer(conf)

    # Subscribe to topic
    consumer.subscribe([topic])

    # Process messages
    try:
        while True:
            msg = consumer.poll(1.0)
            if msg is None:
                # No message available within timeout.
                # Initial message consumption may take up to
                # `session.timeout.ms` for the consumer group to
                # rebalance and start consuming
                print("Waiting for message or event/error in poll()")
                continue
            elif msg.error():
                print('error: {}'.format(msg.error()))
            else:
                # Check for Kafka message
                record_key = "Null" if msg.key() is None else msg.key().decode('utf-8')
                record_value = msg.value().decode('utf-8')
                print("Consumed record with key "+ record_key + " and value " + record_value)
    except KeyboardInterrupt:
        pass
    finally:
        print("Leave group and commit final offsets")
        consumer.close()

```

4. Run the code on the terminal(from the same directory *wd*) follows 
```
python Consumer.py
```
5. You should see the messages as shown below. Note when we produce message from OCI Web Console(as described above in first step), the Key for each message is *Null*
```
$:/path/to/directory/wd>python Consumer.py 
Waiting for message or event/error in poll()
Waiting for message or event/error in poll()
Consumed record with key messageKey0 and value messageValue0
Consumed record with key messageKey1 and value messageValue1
Consumed record with key Null and value Example test message

```

## Next Steps
Please refer to

 1. [Github for Confluent Python SDK](https://github.com/confluentinc/confluent-kafka-python)
 2. [Kafka Python Client](https://docs.confluent.io/clients-confluent-kafka-python/current/overview.html#:~:text=Kafka%20Python%20Client%20Confluent%20develops%20and%20maintains%20confluent-kafka-python%2C,brokers%20%3E%3D%20v0.8%2C%20Confluent%20Cloud%20and%20Confluent%20Platform.)
