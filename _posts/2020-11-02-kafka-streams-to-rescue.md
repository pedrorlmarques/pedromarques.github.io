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


![alt text](/assets/img/Workflow-1.png "Product Activation Workflow")

Now that we understand where the messages came from, we can now start to explain what the requirement was.

## Requirement

The requirement is, “Once a product activation message arrives to service A, it should send the message to service B and then service B should make a REST call to an external service to get all information. Remarks: You can only call the external service one time for each product activation message.”.

![alt text](/assets/img/Workflow-2.png "Workflow once a product activation arrives")

 
## Problem

In order to make the REST call we need a specific ID, and that ID doesn’t come in the product activation Message so now we have a problem.

After internal research in the “company” we found where to get this information.
It turns out that the topic product information has it.


## Solution 

We started to evaluate this new information, and we came up with a naïve approach which was:

- “Okay, so we know that the product information message triggers the product activation workflow which means that we will always receive the product information first and the product activation after, so this is easy. Let’s start consuming messages from the product information and from product activation and store them in a database. If we receive a product activation, we should check if we already received a product information message. If so, we send the notification message to service B and if not, we store in the database and wait for …”

![alt text](/assets/img/Workflow-3.png "Workflow of consuming two topics and storing them in a database")
