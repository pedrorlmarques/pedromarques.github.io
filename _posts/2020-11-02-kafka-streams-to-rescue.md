---
layout: post
title: Kaka Streas to Rescue
subtitle: Success case
tags: [kafka, kafka streams]
comments: true
---

In my current project we are using Kafka as our message broker, and recently we’ve come across with a problem where we were consuming messages from a topic and the messages didn’t get all the information that we needed to fulfill a requirement. 
But after some research we knew that we could get this extra information from another topic.

Of course, because of privacy concerns I can’t go into specific details, but I’ll explain the problem in a generic way.

Before I start to explain what, our approach was, it’s relevant to explain the workflow involved.

## Workflow

The workflow that I’m going to explain addresses the product activation.

Before a product activation, a message arrives to our backend service and we have two relevant services, each one with a specific task.
 
-	The first service is responsible for receiving an “unknown message” and convert that into a product information message. This message is very important because it triggers a lot of sub-workflows, one of them being the product activation workflow which we are talking right now.

-	The second service, once it receives the product information message, triggers a product activation task and once that is finished it sends an event informing that a specific product was active. 


![alt text](/assets/img/Workflow-1.png "Product Activation Workflow"){: .mx-auto.d-block :}

Now that we understand where the messages came from, we can now start to explain what the requirement was.

## Requirement

The requirement is, “Once a product activation message arrives to service A, it should send the message to service B and then service B should make a REST call to an external service to get all information. Remarks: You can only call the external service one time for each product activation message.”.

![alt text](/assets/img/Workflow-2.png "Workflow once a product activation arrives"){: .mx-auto.d-block :}

 
## Problem

In order to make the REST call we need a specific ID, and that ID doesn’t come in the product activation Message so now we have a problem.

After internal research in the “company” we found where to get this information.
It turns out that the topic product information has it.


## Solution 

We started to evaluate this new information, and we came up with a naïve approach which was:

- “Okay, so we know that the product information message triggers the product activation workflow which means that we will always receive the product information first and the product activation after, so this is easy. Let’s start consuming messages from the product information and from product activation and store them in a database. If we receive a product activation, we should check if we already received a product information message. If so, we send the notification message to service B and if not, we store in the database and wait for …”

![alt text](/assets/img/Workflow-3.png "Workflow of consuming two topics and storing them in a database"){: .mx-auto.d-block :}

When we started to implement this solution, we started to find a lot of problems that in the beginning we weren’t aware of.

-	Sometimes both messages arrived at the same time.

-	If we receive the product information message and then product activation message, a message is sent to Service B and a call to the external service is made. Now, if a product information is received again (an Update to the Product) another message to Service B is sent and another REST call to the external service B is made. This isn’t correct considering what was the requirement.


For the first problem, we thought about putting a synchronized clause around the query to the database to preserve the order of the messages arrived. 

For the second problem we thought about controlling this on the Service B, so if we already received a message, we will not make another call.

We weren’t happy with the solutions we thought of, so we decided to dig into a more specific solution, and we came up with using Kafka Streams.

To understand the solution there are some important concepts that are required to know and I’ll explain them.


## Abstractions

Kafka Streams gives us the possibility to join n messages from n different topics into one. 
There are three abstractions:

-	**KTable** – Similar to a database table, it contains only the last message received on the topic. It should be used when reading from a topic that is log compacted.

-	**KStream** - Similar to a log, it contains all the inserted messages and it is an unbounded data stream. It should be used when reading from a topic that isn’t compacted.

-	**GlobalKTable** - A special table type, where you get data from all partitions of an input topic, regardless of the instance that it is running. By contrast, a KTable only gives you data from the respective partitions of the topic that the instance is consuming from.

## Join

A join operation merges two input streams and/or tables based on the keys of their data records and yields a new stream/table.

## Topology

A topology is a graph of stream processors (nodes) that are connected by streams (edges) or shared state stores. 

There are two special processors in the topology:

-	**Source Processor** - A source processor is a special type of stream processor that does not have any upstream processors. It produces an input stream to its topology from one or multiple Kafka topics by consuming records from these topics and forward them to its down-stream processors.

-	**Sink Processor**- A sink processor is a special type of stream processor that does not have down-stream processors. It sends any received records from its up-stream processors to a specified Kafka topic.

So a stream processor is a node in the processor topology, it represents a processing step to transform data in streams by receiving one input record at a time from its upstream processors in the topology, applying its operation to it, and may subsequently producing one or more output records to its downstream processors.

## Real Solution

![alt text](/assets/img/Workflow-4.png "Workflow consuming two topics using Kafka Streams"){: .mx-auto.d-block :}

Before we started to implement this solution, we checked the topics specification. Both topics were marked as **compacted** and both had the same number of partitions. Having the **same number of partitions** means that they are **co-partitioned.**

The next step was to define to the topology.

![alt text](/assets/img/join_messge.png "Stream Topology created"){: .mx-auto.d-block :}

The topology is composed by **2 source processors and 1 sink processor.**

-	**Product Activation Topic** and **Product Information** Topic are the source processors
-	**Activation-Information Topic** is the sink processor.
Being both marked as compacted means that this topology will have a join of type **KTable – KTable.**

Having all this information the solution was to straight forward.

-	Create the **KTable abstraction** for the **source topics** and define the respective **Serdes**.
-	Create the Join between the two **KTables**.
-	Create the **Value Joiner** containing the logic to enrich the product activation message with the missing ID field.
-	Send the **enriched message to the sink topic.**


```java
Properties properties = new Properties();
properties.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-starter-app");
properties.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
properties.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
properties.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

var streamsBuilder = new StreamsBuilder();

//Source Processors
var productActivation = streamsBuilder.table("product-activation", Consumed.with(Serdes.String(), Serdes.String()));
var productInformation = streamsBuilder.table("product-information", Consumed.with(Serdes.String(), Serdes.String()));

//Join
var join = productActivation.join(productInformation, (pA, pI) -> "PA: " + pA + "PI: " + pI);

//Sink Processor
join.toStream().to("activation-information ", Produced.with(Serdes.String(), Serdes.String()));
```

##Conclusion

At the beginning, we started with a naïve approach which gave us several problems to consider. Luckily, this was a typical use case where Kafka Streams fits very well. Also, since both topics were co-partitioned, otherwise the join wouldn’t be doable and the solution could fail, and both were compacted.

With this solution it was possible to guarantee that the Product Information is received before the Product Activation. Whereas when we had the database, the time between checking if the message is present and actually storing it created a gap where this assertion wasn’t valid. Additionally, we went from “at least once” to “exactly once”.

Not, we just got rid of the synchronized clause and the database, we also created a highly scalable, fault tolerant solution and its processing guarantees are built on top of functionality provided by Kafka storage and messaging layer.
