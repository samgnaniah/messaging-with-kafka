# Messaging with Kafka

Kafka is a distributed, partitioned, replicated commit log service. It provides the functionality of a publish-subscribe messaging system with a unique design. Kafka mainly operates based on a topic model. A topic is a category or feed name to which records get published. Topics in Kafka are always multi-subscriber.

> This guide walks you through the process of messaging with Apache Kafka using Ballerina language. 

The following are the sections available in this guide.

- [What you'll build](#what-you-build)
- [Prerequisites](#pre-req)
- [Developing the service](#developing-service)
- [Testing](#testing)
- [Deployment](#deploying-the-scenario)
- [Observability](#observability)

## <a name="what-you-build"></a>  What you’ll build
To understand how you can use Kafka for publish-subscribe messaging, let's consider a real-world use case of a product management system. This product management system consists of a product admin portal using which the product administrator can update the price for a product. This price update message should be consumed by a couple of franchisees and an inventory control system to take appropriate actions. Kafka is an ideal messaging system for this scenario. In this particular use case, once the admin updates the price of a product, the update message is published to a Kafka topic called 'product-price' to which the franchisees and the inventory control system subscribed to listen. The following diagram illustrates this use case clearly.


![alt text](/images/messaging-with-kafka.svg)


In this example, the Ballerina Kafka Connector is used to connect Ballerina to Apache Kafka. With this Kafka Connector, Ballerina can act as both message publisher and subscriber.

## <a name="pre-req"></a> Prerequisites
- JDK 1.8 or later
- [Ballerina Distribution](https://github.com/ballerina-lang/ballerina/blob/master/docs/quick-tour.md)
- [Apache Kafka 1.0.0](https://kafka.apache.org/downloads)
  * Download the binary distribution and extract the contents
- [Ballerina Kafka Connector](https://github.com/wso2-ballerina/package-kafka)
  * After downloading the ZIP file, extract it and copy the containing JAR files into the <BALLERINA_HOME>/bre/lib folder
- A Text Editor or an IDE 

**Optional requirements**
- Ballerina IDE plugins ([IntelliJ IDEA](https://plugins.jetbrains.com/plugin/9520-ballerina), [VSCode](https://marketplace.visualstudio.com/items?itemName=WSO2.Ballerina), [Atom](https://atom.io/packages/language-ballerina))
- [Docker](https://docs.docker.com/engine/installation/)

## <a name="developing-service"></a> Developing the service

### <a name="before-begin"></a> Before you begin
##### Understand the package structure
Ballerina is a complete programming language that can have any custom project structure that you wish. Although the language allows you to have any package structure, use the following package structure for this project to follow this guide.

```
messaging-with-kafka
├── ProductMgtSystem
│   ├── Publisher
│   │   ├── product_admin_portal.bal
│   │   └── product_admin_portal_test.bal
│   └── Subscribers
│       ├── Franchisee1
│       │   └── franchisee1.bal
│       ├── Franchisee2
│       │   └── franchisee2.bal
│       └── InventoryControl
│           └── inventory_control_system.bal
└── README.md

```

Package `Publisher` contains the file that handles the Kafka message publishing and a unit test file. 

Package `Subscribers` contains three different subscribers who subscribed to Kafka topic 'product-price'.


### <a name="Implementation"></a> Implementation

Let's get started with the implementation of a Kafka service, which is subscribed to the Kafka topic 'product-price'. Let's consider `inventory_control_system.bal` for example. Let's first see how to add the Kafka configurations for a Kafka subscriber written in Ballerina language. Refer to the code segment attached below.

##### Kafka subscriber configurations
```ballerina
// Kafka subscriber configurations
@Description {value:"Service level annotation to provide Kafka consumer configuration"}
@kafka:configuration {
    bootstrapServers:"localhost:9092, localhost:9093",
    // Consumer group ID
    groupId:"inventorySystem",
    // Listen from topic 'product-price'
    topics:["product-price"],
    // Poll every 1 second
    pollingInterval:1000
}
```

A Kafka subscriber in Ballerina should contain the `@kafka:configuration {}` block in which you specify the required configurations for a Kafka subscriber. 

The `bootstrapServers` field provides the list of host and port pairs, which are the addresses of the Kafka brokers in a "bootstrap" Kafka cluster. 

The `groupId` field specifies the Id of the consumer group. 

The `topics` field specifies the topics that must be listened by this consumer. 

The `pollingInterval` field is the time interval that a consumer polls the topic. 

Let's now see the complete implementation of the `inventory_control_system.bal` file, which is a Kafka topic subscriber.

##### inventory_control_system.bal
```ballerina
package ProductMgtSystem.Subscribers.InventoryControl;

import ballerina.net.kafka;
import ballerina.log;

// Kafka subscriber configurations
@Description {value:"Service level annotation to provide Kafka consumer configuration"}
@kafka:configuration {
    bootstrapServers:"localhost:9092, localhost:9093",
    // Consumer group ID
    groupId:"inventorySystem",
    // Listen from topic 'product-price'
    topics:["product-price"],
    // Poll every 1 second
    pollingInterval:1000
}
// Kafka service that listens from the topic 'product-price'
// 'inventoryControlService' is subscribed to the new product price updates from the product admin and updates the database
service<kafka> inventoryControlService {
    // Triggered whenever a message is added to the subscribed topic
    resource onMessage (kafka:Consumer consumer, kafka:ConsumerRecord[] records) {
        // Dispatch a set of Kafka records to service and process one after another
        int counter = 0;
        while (counter < lengthof records) {
            // Get the serialized message
            blob serializedMsg = records[counter].value;
            // Convert the serialized message to a string message
            string msg = serializedMsg.toString("UTF-8");
            log:printInfo("New message received from the product admin");
            // Log the retrieved Kafka record
            log:printInfo("Topic: " + records[counter].topic + "; Received Message: " + msg);
            // Mock logic
            // Update the database with the new price for the specified product
            log:printInfo("Database updated with the new price for the specified product");
            counter = counter + 1;
        }
    }
}

```

In the above code, you have implemented a Kafka service that is subscribed to listen to the 'product-price' topic. You provided the Kafka subscriber configurations for this Kafka service as shown above. 

Resource `onMessage` is triggered whenever a message is published to the topic specified.

To check the implementations of the other subscribers, see the [franchisee1.bal](https://github.com/ballerina-guides/messaging-with-kafka/blob/master/ProductMgtSystem/Subscribers/Franchisee1/franchisee1.bal) file and the [franchisee2.bal](https://github.com/ballerina-guides/messaging-with-kafka/blob/master/ProductMgtSystem/Subscribers/Franchisee2/franchisee2.bal) file.

Let's next focus on the implementation of `product_admin_portal.bal`, which acts as the message publisher. It contains an HTTP service, using which, a product admin can update the price of a product. 

In this example, you first serialized the message in `blob` format before publishing it to the topic. Then `kafka:ProducerRecord` is created where you specify the serialized message, destination topic name, and the number of partitions. Then you specified the Kafka producer cofigurations by creating a `kafka:ProducerConfig`. Look at the code snippet added below.

##### Kafka producer configurations
```ballerina
// Kafka message publishing logic
// Construct and serialize the message to be published to the Kafka topic
json priceUpdateInfo = {"Product":productName, "UpdatedPrice":newPrice};
blob serializedMsg = priceUpdateInfo.toString().toBlob("UTF-8");
// Create the Kafka ProducerRecord and specify the destination topic - 'product-price' in this case
// Set a valid partition number, which will be used when sending the record
kafka:ProducerRecord record = {value:serializedMsg, topic:"product-price", partition:0};

// Create a Kafka ProducerConfig with optional parameters 
// 'clientID' is used for broker side logging,
// acks refers to the number of acknowledgments for requests 
// noRetries refers to the number of retries if the record fails to get sent
kafka:ProducerConfig producerConfig = {clientID:"basic-producer", acks:"all", noRetries:3};
// Produce the message and publish it to the Kafka topic
```

Let's now see the structure of the `product_admin_portal.bal` file. Inline comments are added for better understanding.

##### product_admin_portal.bal
```ballerina
package ProductMgtSystem.Publisher;

// imports

// Constants to store admin credentials
const string ADMIN_USERNAME = "Admin";
const string ADMIN_PASSWORD = "Admin";

// Product admin service
@http:configuration {basePath:"/product"}
service<http> productAdminService {
    // Resource that allows the admin to send a price update for a product
    @http:resourceConfig {methods:["POST"], consumes:["application/json"], produces:["application/json"]}
    resource updatePrice (http:Connection connection, http:InRequest request) {
      
        // Try getting the JSON payload from the incoming request

        // Check whether the specified value for 'Price' is appropriate

        // Check whether the credentials provided are Admin credentials

        // Kafka message publishing logic
        // Construct and serialize the message to be published to the Kafka topic
        json priceUpdateInfo = {"Product":productName, "UpdatedPrice":newPrice};
        blob serializedMsg = priceUpdateInfo.toString().toBlob("UTF-8");
        // Create the Kafka ProducerRecord and specify the destination topic - 'product-price' in this case
        // Set a valid partition number, which will be used when sending the record
        kafka:ProducerRecord record = {value:serializedMsg, topic:"product-price", partition:0};

        // Create a Kafka ProducerConfig with optional parameters 
        // 'clientID' is used for broker side logging
        // acks refers to the number of acknowledgments for requests 
        // noRetries refers to the number of retries if a record fails to get sent
        kafka:ProducerConfig producerConfig = {clientID:"basic-producer", acks:"all", noRetries:3};
        // Produce the message and publish it to the Kafka topic
        kafkaProduce(record, producerConfig);
        
        // Send a success status to the admin request
    }
}

// Function to produce and publish a given record to a Kafka topic
function kafkaProduce (kafka:ProducerRecord record, kafka:ProducerConfig producerConfig) {
    // Kafka ProducerClient endpoint
    endpoint<kafka:ProducerClient> kafkaEP {
        create kafka:ProducerClient(["localhost:9092, localhost:9093"], producerConfig);
    }
    // Publish the record to the specified topic
    kafkaEP.sendAdvanced(record);
    kafkaEP.flush();
    // Close the endpoint
    kafkaEP.close();
}

```

To see the complete implementation of the above, see the [product_admin_portal.bal](https://github.com/ballerina-guides/messaging-with-kafka/blob/master/ProductMgtSystem/Publisher/product_admin_portal.bal) file. 

## <a name="testing"></a> Testing 

### <a name="try-it"></a> Try it out

1. Start the `ZooKeeper` instance with default configurations by entering the following command in a terminal.

   ```bash
   <KAFKA_HOME_DIRECTORY>$ bin/zookeeper-server-start.sh config/zookeeper.properties
   ```

2. Start a single `Kafka broker` instance with default configurations by entering the following command in a different terminal.

   ```bash
   <KAFKA_HOME_DIRECTORY>$ bin/kafka-server-start.sh config/server.properties
   ```
   Here we started the Kafka server on host:localhost, port:9092. Now we have a working Kafka cluster.

3. Create a new topic `product-price` on Kafka cluster by entering the following command in a different terminal.

   ```bash
   <KAFKA_HOME_DIRECTORY>$ bin/kafka-topics.sh --create --topic product-price --zookeeper localhost:2181 --replication-factor 1 --partitions 2
   ```
   Here we created a new topic that consists of two partitions with a single replication factor.
   
4. Run the `productAdminService`, which is an HTTP service that publishes messages to the Kafka topic, and the Kafka services in the `Subscribers` package, which are subscribed to listen to the Kafka topic by entering the following commands in sperate terminals.

   ```bash
    <SAMPLE_ROOT_DIRECTORY>$ ballerina run ProductMgtSystem/Publisher/
   ```

   ```bash
    <SAMPLE_ROOT_DIRECTORY>$ ballerina run ProductMgtSystem/Subscribers/<Subscriber_Package_Name>/
   ```
   
5.  Invoke the `productAdminService` by sending a POST request to update the price of a product with Admin credentials.

    ```bash
    curl -v -X POST -d '{"Username":"Admin", "Password":"Admin", "Product":"ABC", "Price":100.00}' \
     "http://localhost:9090/product/updatePrice" -H "Content-Type:application/json"
    ```

    The `productAdminService` sends a response similar to the following:
    ```bash
     < HTTP/1.1 200 OK
    {"Status":"Success"}
    ```

    Sample log messages in subscribed Kafka services:
    ```bash
     INFO  [ProductMgtSystem.Subscribers.<All>] - New message received from the product admin 
     INFO  [ProductMgtSystem.Subscribers.<All>] - Topic: product-price; Received Message {"Product":"ABC","UpdatedPrice":100.0} 
     INFO  [ProductMgtSystem.Subscribers.Franchisee1] - Acknowledgement from Franchisee 1 
     INFO  [ProductMgtSystem.Subscribers.Franchisee2] - Acknowledgement from Franchisee 2 
     INFO  [ProductMgtSystem.Subscribers.InventoryControl] - Database updated with the new price for the specified product
    ```

### <a name="unit-testing"></a> Writing unit tests 

In Ballerina, the unit test cases should be in the same package and the naming convention should be as follows.
* Test files should contain _test.bal suffix.
* Test functions should contain test prefix.
  * e.g., testProductAdminService()

This guide contains a unit test case for the HTTP service `productAdminService` from the `product_admin_portal.bal` file. The test file is in the same package that the above-mentioned file is located.

To run the unit test, go to the sample root directory and run the following command.
   ```bash
   <SAMPLE_ROOT_DIRECTORY>$ ballerina test ProductMgtSystem/Publisher/
   ```

To check the implementation of this test file, see the [product_admin_portal_test.bal](https://github.com/ballerina-guides/messaging-with-kafka/blob/master/ProductMgtSystem/Publisher/product_admin_portal_test.bal) file.

## <a name="deploying-the-scenario"></a> Deployment

Once you are done with the development, you can deploy the service using any of the methods listed below. 

### <a name="deploying-on-locally"></a> Deploying locally
You can deploy the services that you developed above in your local environment. You can create the Ballerina executable archives (.balx) first and run them in your local environment as follows.

Building 
   ```bash
    <SAMPLE_ROOT_DIRECTORY>$ ballerina build ProductMgtSystem/Publisher/

    <SAMPLE_ROOT_DIRECTORY>$ ballerina build ProductMgtSystem/Subscribers/<Subscriber_Package_Name>/

   ```

Running
   ```bash
    <SAMPLE_ROOT_DIRECTORY>$ ballerina run <Exec_Archive_File_Name>

   ```

### <a name="deploying-on-docker"></a> Deploying on Docker
(Work in progress) 

### <a name="deploying-on-k8s"></a> Deploying on Kubernetes
(Work in progress) 


## <a name="observability"></a> Observability 

### <a name="logging"></a> Logging
(Work in progress) 

### <a name="metrics"></a> Metrics
(Work in progress) 


### <a name="tracing"></a> Tracing 
(Work in progress) 


## P.S.

Due to an [issue](https://github.com/wso2-ballerina/package-kafka/issues/2), Ballerina Kafka Connector does not work with Ballerina versions later than 0.96.0 (exclusive). Therefore, when trying this guide use Ballerina version 0.96.0.
