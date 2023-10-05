# Eventual Consistency and you! A.K.A. How Mold-E Event Consumption works!

## Assumed Knowledge
This article assumes you're aware of event driven applications that consume events from a Message Broker ( Memphis, Apache Kafka/ActiveMq, RabbitMQ etc. ).

## From the Wikipedia article:
"Eventual consistency is a consistency model used in distributed computing to achieve high availability that informally guarantees that, if no new updates are made to a given data item, eventually all accesses to that item will return the last updated value"
To summarize for our use, it would mean that after consuming all of the events that change a data item, no matter what order they are emitted in, or consumed, we should always arrive at the same value after the consumption has completed. This is EXTREMELY important for the functionality of Core Services, and the procedure that we use to repopulate our database by replaying all events through Core Services again.

## The Problem
When we are working with an event stream, we will have different types of events being consumed, that are going to be processed with varying results. When working with an incrementing value, then as long as you are incrementing the same accumulator for each related event, you will eventually ( after consuming all events ) be consistent. Unfortunately not all events that are processed can be broken down into simple accumulators. This article will discuss the specific problem that can occur when a record is updated with non-accumulator values.  

Let's break down this concept with an illustrated basic event processing solution. 

![Figure 1](./images/Eventual%20Consistency%20Fig%201.png)
<div style="text-align: center;">Figure 1.</div>

In figure 1, we are assuming use of a Message Broker, and an event consumer application ( here we refer to it as _Event Processor_ ) that is consuming from a queue on the Message Broker.
In the above example, we have 20 events in a queue that we will be consuming. For our explanation, we will be focusing on the event stream regarding **User A**, which has a creation event at position 5, and two updates at positions 9 and 11.

In figure 1, we assume to only have 1 worker that is processing the messages in the _Event Processor_, with a prefetch set to 10, meaning we will grab 10 events when pulling from the Message Broker. Once it has completed processing these 10 events ( 1-10 ), the _Event Processor_ will then retrieve the next 10 events ( 11-20 ) to process, completing the example events in the queue. 

In this example, it's very simple to attain consistency. The shown configuration will be able to process all of these events in the order that they were emitted. This however, is extremely limited in processing speed. It is not horizontally scalable in this design.

Let's look at another solution, which is horizontally scalable. Here we will add one more worker to the solution, and immediately we can illustrate the issue.

![Figure 2](./images/Eventual%20Consistency%20Fig%202.png)
<div style="text-align: center;">Figure 2.</div>

In this configuration, **Worker 1** will prefetch events 1-10 and start processing them, while in parallel **Worker 2** will prefetch events 11-20 and start processing those. 
This means the order of events being processed can very loosely be considered as: 1 and 11 consumed at the same time, 2 and 12, 3 and 13, etc. 

This translates to the order of the **User A** events that we are interested it would be consumed in the order: `[11, 5, 9]`. Let's go ahead and give these events more definition to more clearly illustrate the issue:

### User A Create - Event Position 5
```
Event Payload {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clafave'
    fullName: 'Cris Lafave'
    organization: 'Engineering'
    datecode: 10000000
}
```
### User A Update - Event Position 9
```
Event Payload {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clafav'
    fullName: 'Cris Lefeve'
    organization: 'Engineering'
    datecode: 20000000
}
```
### User A Update - Event Position 11
```
Event Payload {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clefevre'
    fullName: 'Christopher Lefevre'
    organization: 'Engineering'
    datecode: 30000000
}
```
With the events defined above, if following the order properly, we end up with the following resulting record:

**Resulting Record**
```
User Record {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clefevre',
    fullName: 'Christopher Lefevre',  
}
```

However, in the solution shown in Figure 2, we see that User A Update - Event 9 would be the final update processed ( with 2 event processing workers ) and instead of the CORRECT final update we will end up with the result below:

**Resulting Record**
```
User Record {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clafav',
    fullName: 'Cris Lefevre',  
}
```

## The Solution
One of the simplest solutions to this issue is to only allow the latest event to update the data item ( The User record ). In order to persist only the latest event data, we can add a new field to the data item that would track the date of latest update. We would set this field based on the timestamp included in the event ( For our purposes we have `datecode` in the payload ). When processing the event stream, we would then need to check the `datecode` value of the event payload, against the `lastUpdate` value in the record, only allowing the update to proceed if the event's `datecode` is a later date than the `lastUpdate` field.

With the configuration from Figure 2, we would be consuming the events in the order [5,11,9]. In reality, we could consume these in any order, and after every event is consumed, we can assure that we are _eventually consistent!_

### Processing Event 5:
**Event Data**
```
Event Payload {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clafave'
    fullName: 'Cris Lafave'
    organization: 'Engineering'
    datecode: 10000000
}
```
**Resulting Record**
```
User Record {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clafave',
    fullName: 'Cris Lafave',
    last-update: 10000000
}
```

Since this is the first event processed, there is no `lastUpdate` value to compare against, therefore it automatically is accepted as being the latest date as we currently have nothing in the database for this `userId`!

### Processing Event 11: 
**Event Data**
```
Event Payload {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clefevre'
    fullName: 'Christopher Lefevre'
    organization: 'Engineering'
    datecode: 30000000
}
```
**Resulting Record**
```
User Record {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clefevre',
    fullName: 'Christopher Lefevre',  
    last-update: 30000000
}
```
Before updating the data, we need to check our new last-update value ( 30000000 ) against the existing records last-update value ( 10000000 ) which was set by the previous event. We see that our value is greater ( 30000000 > 10000000 ), so the data item IS updated.

### Processing Event 9:
**Event Data**
```
Event Payload {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clafav'
    fullName: 'Cris Lefeve'
    organization: 'Engineering'
    datecode: 20000000
}
```
**Resulting Record**
```
User Record {
    userId: 93fd7598-bee1-4596-9b6e-32145c18c35f,
    login: 'clefevre',
    fullName: 'Christopher Lefevre',  
    last-update: 30000000
}
```
Now we again check to see if the `lastUpdate` value that we have ( `20000000` ) against the existing records `lastUpdate` value ( `30000000` ), which was set by the previous event ( 9 ). We now see that our value is NOT greater ( `20000000 < 30000000` ), so the data record is NOT updated.

We are then left with a system that will show that we are eventually consistent and for this record we will always keep only the most up to date information, instead of simply the last event that happened to be processed!