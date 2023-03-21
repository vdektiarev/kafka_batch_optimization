<a id="introduction"></a>
# More than One Way to Update your Kafka-Backed Entity from another Data Source using Kafka Streams 

Data update processes can sometimes be complicated in event-driven ecosystems. 

In the most simple case, if you have a business entity which needs to be kept updated, and you have a process generating updates for this specific entity, it's of course more than straightforward.
Let's say we have Kafka as a backbone for our events' storage and an entity the state of which is tracked there (topic with messages - key is the entity unique identifier, value is its state). If we have a Kafka producer for this topic, the updates are easily implemented, as each new message is basically an update.

However, if the things are not that simple, and we have a separate datasource and separate processes which track some of the entity's attribute values, and we need to integrate this part of the system with our main Kafka system somehow, there are ways to do this, and you need to choose them wisely.

In this article we are going to review one real-world scenario when an existing Kafka ecosystem with Kafka Streams under the hood had to adopt a small business requirement resulting in a need to automate updates of an entity from a totally separate business process not related to Kafka at all.

<a id="requirements"></a>
## Business Requirements and Constraints

We are dealing with an eCommerce platform, which uses Kafka as a backbone for processing items updates which are sold in the online shop.



<a id="existing-setup"></a>
## Existing Setup

<a id="option-1"></a>
## Option 1: stream the data flow to Kafka and perform a Foreign-Key Join

<a id="option-2"></a>
## Option 2: Use Kafka Streams and Update Commands to merge the datasources

<a id="comparison-final-thoughts"></a>
## Comparison and Final Thoughts

