<!-- TOC -->
* [More than One Way to Update your Kafka-Backed Entity from another Data Source using Kafka Streams](#more-than-one-way-to-update-your-kafka-backed-entity-from-another-data-source-using-kafka-streams)
  * [Business Context, Requirements and Constraints](#business-context-requirements-and-constraints)
  * [Existing Setup](#existing-setup)
  * [Option 1: Write the Data to Kafka and Perform a Foreign-Key Join](#option-1--write-the-data-to-kafka-and-perform-a-foreign-key-join)
  * [Option 2: Use Kafka Streams and Update Commands to merge the datasources](#option-2--use-kafka-streams-and-update-commands-to-merge-the-datasources)
  * [Comparison and Final Thoughts](#comparison-and-final-thoughts)
<!-- TOC -->

# More than One Way to Update your Kafka-Backed Entity from another Data Source using Kafka Streams 

Data update processes can sometimes be complicated in event-driven ecosystems. 

In the most simple case, if you have a business entity which needs to be kept updated, and you have a process generating updates for this specific entity, it's of course more than straightforward.
Let's say we have Kafka as a backbone for our events' storage and an entity the state of which is tracked there (topic with messages - key is the entity unique identifier, value is its state). 
If we have a Kafka producer for this topic, the updates are easily implemented, as each new message is basically an update.

However, if the things are not that simple, and we have a separate datasource and separate processes which track some of the entity's attribute values, and we need to integrate this part of the system with our main Kafka system somehow, there are ways to do this, and you need to choose them wisely.

In this article we are going to review one real-world scenario when an existing Kafka ecosystem with Kafka Streams under the hood had to adopt a small business requirement resulting in a need to automate updates of an entity from a totally separate business process not related to Kafka at all.

## Business Context, Requirements and Constraints

We are dealing with an eCommerce platform, which sells items via online shop. 
But before the items information becomes available to the customers, this information needs some processing: ingestion from the suppliers' data stores, rules applying, categorization, data transformation etc. 
This is applied to lots of item attributes like tags, prices, stock information, description, size etc. 
Once it's done, the items become visible at the online shop website.

One of the attributes tells the estimate minimum and maximum time when the item could be delivered to the customer once ordered. 
We will call it "delivery time" from now on. 
Logistics department manually controls these delivery times by assigning minimum and maximum delivery time to each "delivery class" - and every item has a delivery class assigned.

![Delivery Class Table](images\delivery_class_table.png)

They update delivery times via their own specific process they are used to, and the data is saved to their own separate datasource which has nothing to do with the overall existing items data processing and storage platform.
Once they change the time for a specific class, they expect every item which has this class assigned to eventually have the updated delivery time values, and these values should be visible in the item description page in the online shop.

Items information is supposed to be visible not only in the online shop, but also at other different data integrations, like Google searches, partner shops etc. 
So it's not acceptable to assign updated delivery time values to the item at the very end stage of items information processing, when the item state lands to the shop datasource. 
This needs to be processed before the information gets distributed to other channels.

## Existing Setup

The shop uses Kafka as a backbone for processing items' state update events. 
The data is processed with applications using Kafka Streams library, and then stored in different data stores including ones exposed via API to the actual shop frontend. 
All data transformations, aggregations, manipulations are happening within Kafka topics, and delivery times are assigned to the items at early stages of items data aggregation into one data object - before items are classified into categories and distributed across the destinations where the items data is expected.

Delivery times assignment is performed by one of Kafka Streams applications which calculates all stock data. Initial items data is received from the supplier via a Kafka topic as an event identified by item SKU, which is sent by a producer related to the items' supplier.
Once the item data is received, besides all other stock data calculations, the application queries an AWS DynamoDB table which has delivery class -> minimum and maximum delivery time mappings.

Logistics department stores their class -> times mappings in an SQL RDBMS, and they can have a job which they schedule or launch manually to write this data from their tables to basically anywhere.

Below picture shows the concept view of the existing setup which is described above.

![Existing Setup](images\existing_setup.png)

The problem of such an approach is that when the times for a class are changed, and even are propagated to DynamoDB table, it's challenging to update the times values of all items which have this delivery class assigned.
To do that, the item has to be updated via the supplier channel, so that the stream application could pick an event with its new state, go to DynamoDB to look up the times and assign them to the object, and then send the enriched object as a message further.
But what if the supplier doesn't need to update the whole item, and only Logistics department wants to update the delivery times for the item?
In that case someone needs to produce a "fake" update of the whole item through supplier channel to have the times updated. 
And one should not forget to roll back these updates by producing items with their original state through the same channel, so that only delivery times attributes would be updated.
And since there could be thousands of items having delivery class which is changed, one needs to retrieve all these item states from some datastore (and hope it's the latest one and no other updates are pending in the streams pipeline), prepare messages with fake updates and send them through the channel. 
This is a lot of manual work and operational overhead which can bring inconsistencies to the data. 

Business would like to automate the process and remove the overhead + maintain data consistency.
Let's review the possible ways how to solve this problem.

## Option 1: Write the Data to Kafka and Perform a Foreign-Key Join

If we try to consider delivery classes data including times as items related data and a separate business entity, it could be a part of our data processing ecosystem in the form of a separate Kafka topic and corresponding KTable, which then can be used in some business processes.

One of the most classical ways to change an entity's attribute within one topic in Kafka, taking the value from another one, is Kafka Streams Foreign Key Join.
The join happens between two KTables which represent messages aggregation from a given topic. And since Kafka Streams are capable to read streaming events of entity's state changes, we can use them to join two streams according to our business logic and produce the output as aggregated events to the downstream topic.

To implement this approach, we first need to deliver the data from DynamoDB to Kafka. Having our data sourced in DynamoDB, we could have leveraged DynamoDB Streams and, for example, a Lambda function to produce all inserts or updates to the Kafka topic.
But since we'll have our delivery classes information sourced both in Kafka, and in Logistics Department database, we might want to get rid of the additional data source business won't be using.

The algorithm to implement the approach will be the following:

1. Create a new Kafka topic for delivery classes. Choose the number of partitions and replication factor.
2. Create a Producer for the topic. It could be, for example, a REST endpoint which accepts a class as an entity.
3. Perform the initial load of delivery classes into Kafka. Since Logistics department already has a job to post delivery classes data from their database to any destination, we could change it to perform REST HTTP calls to our new endpoint to save classes data into our Kafka topic.
4. Change our items data processing stream topology to perform a Foreign Key Join between the table with items data from supplier and the table with delivery classes data. To do this, you need to implement a ValueJoiner and use it in the topology logic. Write the result to the downstream topic for further processing.

The code for ValueJoiner could look in the following way in Java:

```java
package io.confluent.developer;

import de.soname.developer.avro.Item;
import de.soname.developer.avro.DeliveryClass;
import de.soname.developer.avro.EnrichedItem;
import org.apache.kafka.streams.kstream.ValueJoiner;

public class ItemDeliveryClassJoiner implements ValueJoiner<Item, DeliveryClass, EnrichedItem> {
    public EnrichedItem apply(Item item, DeliveryClass deliveryClass) {
        return EnrichedItem.newBuilder()
                .setId(item.getId())
                .setDescription(item.getDescription())
                .setMinimumDeliveryTime(deliveryClass.getMinimumTime())
                .setMaximumDeliveryTime(deliveryClass.getMaximumTime())
                .build();
    }
}
```

The code for the topology could look like this:

```java

```

The architecture of the target setup would look like this:

![Option 1](images\option1.png)

The solution has the following benefits:

1. This approach naturally uses Kafka and Kafka Streams which aligns conceptually with the overall approach of data processing in the system.
2. The process is automated and does not rely on synthetic updates. One job needs to be executed once the classes are ready to be posted.
3. The approach is cost-effective since it does not need a DynamoDB table and frequent reads from it.
4. 

## Option 2: Use Kafka Streams and Update Commands to merge the datasources

## Comparison and Final Thoughts

