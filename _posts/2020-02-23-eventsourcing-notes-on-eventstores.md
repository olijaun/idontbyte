---
layout: test_post
title:  "Event Sourcing - Notes on Event Stores"
categories: [architecture, design, eventsourcing]
comments: true
---

This is the first part in a series about Event Sourcing. In the past year I was involved in the development of a Java application using Event Sourcing. Actually we did it twice using different approaches. In this post I'd like to share some thoughts about Event Stores.

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).

## What is an Event Store?

Martin Fowler writes about different Event-Driven Patterns in [this article](https://martinfowler.com/articles/201701-event-driven.html?). For example he mentions *Event Notification* which is used to notify other systems of a change. Events in event sourcing might also be used for this purpose but it is not their main purpose.

In an Event Sources System events are primarily used **to reconstitute the state of the system**. These events are stored in an Event Store. 

An Event Store must be capable of storing streams of events. An event stream is just an ordered list of events belonging to an aggregate (in terms of DDD). The events are stored in order they are emitted.

{% figure caption:"Streams in an Event Store" %}
{% graphviz  %}
digraph G {
    rankdir="LR";
    node [fontname=Helvetica, fontsize=10, width = 2];
    graph [ fontname=Helvetica, fontsize=10 ];
    edge [ fontname=Helvetica, fontsize=10 ];
    event31 [shape=record,label="Event #1\nCreated"]
    event32 [shape=record,label="Event #2\nNameChanged"]
    event33 [shape=record,label="Event #3\nMoved"]
    event21 [shape=record,label="Event #1\nCreated"]
    event22 [shape=record,label="Event #2\nJobChanged"]
    event11 [shape=record,label="Event #1\nCreated"]
    event12 [shape=record,label="Event #2\nEmailAddressChanged"]
    event13 [shape=record,label="Event #3\nNameChanged"]
    event14 [shape=record,label="Event #4\nDied"]
	subgraph cluster_0 {
      	label = "Stream for person-A";
       	labeljust="l";
       	color=lightgrey;
       	event11 -> event12 -> event13 -> event14;
    }
    subgraph cluster_1 {
        label = "Stream for person-B";
        labeljust="l";
        color=lightgrey;
        event21 -> event22;
    }
	subgraph cluster_2 {
    	label = "Stream for person-C";
    	labeljust="l";
    	color=lightgrey;
    	event31 -> event32 -> event33;
    }
}
{% endgraphviz %}
{% endfigure %}

An example of an aggregate might be a "Person". There may be multiple Persons *A*, *B*, *C*. For each of these Persons there is a separate stream. The streams might be called *person-A*, *person-B*, *person-C*. The order of the events inside a stream is very important because otherwise there could occur illegal state transitions when replaying.

An Event Store also must provide a way to read events. At least it should be possible to read events by stream/aggregate. In most cases however an Event Store should also be able to read events by type.

When we talk about minimal requirement for an Event Store then I would say that subscriptions/notifications are not a requirement. Subscriptions can be built on top of the reading facility.

# Optimistic Locking

Vaughn Vernon's [IDDD book](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design) and [Greg Young's papers on CQRS](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) both show a similar interface for an Event Store. Here's Greg Young's Version (taken from his [example project](https://github.com/gregoryyoung/m-r/blob/master/SimpleCQRS/EventStore.cs)):

```csharp
public interface IEventStore
{
    void SaveEvents(Guid aggregateId, IEnumerable<Event> events, int expectedVersion);
    List<Event> GetEventsForAggregate(Guid aggregateId);
}
```

`aggregateId` implies Domain Driven Design (DDD) although it is not necessary to use DDD for Event Sourcing but it somehow fits very well. Instead of calling it `aggregateId` it could also be called `eventStreamId` or similar. Also note that this interface only provides `GetEventsForAggregate` because Greg Young's implementation publishes events after persisting them. His implementation is very basic so don't bother about the details here.

However for me the most notable thing is the `expectedVersion`. Greg Young explains in his [his paper](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf), that this is used for optimistic locking. So when saving an aggregate (or stream) the `expectedVersion` indicates the version of the latest event in the given stream expected when appending the new event. If the aggregate has been modified in the meantime then the Event Store will throw a `ConcurrencyException`. This mechanism is important in order to enforce business rules.

Imagine a bank account: If there is a balance of 42 CHF and two people try to withdraw these 42 CHF at almost the same time then only one must succeed.

## Whats wrong with Optimistic Locking?

Optimistic Locking might be a performance issue. But apart from that it can be very inconvenient for the users of the system. Imagine again the *Bank Account Aggregate*: On withdrawal we have to check that there is enough money. This makes sense. But what about depositing money? Does it matter in which order deposits are made? Probably not. So why bother the user with a `ConcurrencyException` or something similar?

It would be simple to support for example `-1` as a value to the `expectedVersion` which would tell the Event Store that it should not check the version (this is actually the way it is implemented in [Greg Young's example](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)). But then how does the repository know in which case the version matters and in which case not? The Repository implementation would need to have business knowledge.

A solution proposed in the [IDDD Book](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design) is that the aggregate is reloaded in case of a `ConcurrencyException` and the command is retried. This can be repeated until the command  either succeeds or fails with an actual business exception.

The problem that Optimistic Locking solves is technical. For business people there is always just one instance of a given aggregate (account-123, person-A, etc.). However for technical reasons there might be multiple writers due to concurrent invocations of an operation on the same aggregate.

All other motivations for optimistic locking are actually business requirements. For example the business people might like to avoid that a Person's name can be overwritten by a user shortly after is has been changed by another user. Although Optimistic Locking could be used to solve this, it would be more adequate to implement this as a business rule instead of "abusing" a technical solution for this purpose. 

## Single Writer

There is no need for Optimistic Locking if there is always only a single writer per aggregate. This is not simple to achieve if you have a clustered application.

One could use an Actor Frameworks like Akka or Vlingo which provide clustering facilities. I will go into more details on how this could be implemented in a later post. Actors basically have an inbox where they receive messages (e.g. a Command) and process them one by one. If the system is deployed on multiple nodes then sharding is required. Each shard acts as the single source for a set of aggregates. This setup assures that there is always a single writer and no Optimistic Locking is required.

# Implementation challenges

Greg young writes:

> Although not a trivial exercise to create a production quality Event Storage the overall concepts behind
  an Event Storage are relatively easy
  
Why shouldn't this be trivial? Well, one thing is getting the optimistic locking right.

Another challenge I came across was when I tried to implement my own read journal for Akka. A Read Journal is basically the "Read Side" of the EventStore. Usually you want to subscribe to events in an Event Store. This could be a subscription to a specific aggregate, aggregate type or to specific event types.

Events are used to update projections (aka Read Model) among other things. A projection could be rebuilt from scratch each time the system is started. This could get slow if there are many events in the system. Another approach is to have a persistent projection that always stores the event number of the last event that has been processed. 

Let's say the system was shutdown after having read events up to number 2567. When restarting the system the Projection requests all events after 2567. If the projection subscribes to one single stream/aggregate then it is simple because if Optimistic Locking or a Single Writer is used then the aggregate's sequence numbers are always continuous and do not have gaps. After event number *n* there must follow event number *n + 1* inside the same stream/aggregate.

But what happens if the projection listens to a set of event types emitted by different aggregates types? The simplest approach to this is to have an auto increment field/column that provides a unique global sequence number across all streams/aggregates.

<table border='1' cellborder='0' style='rounded' width='100%'>
    <thead>
       <tr>
           <th>Global Seq</th>
           <th>Stream</th>
           <th>Seq</th>
           <th>EventType</th>
           <th>EventData</th>
       </tr>
    </thead>
    <tbody>
        <tr>
            <td>1</td>
            <td>person-1</td>
            <td>1</td>
            <td>Created</td>
            <td>{ ...}</td>
        </tr>
        <tr>
            <td>2</td>
            <td>person-1</td>
            <td>2</td>
            <td>EmailAddressChanged</td>
            <td>{ ...}</td>
        </tr>
        <tr>
            <td>3</td>
            <td>person-2</td>
            <td>1</td>
            <td>Created</td>
            <td>{ ...}</td>
        </tr>
        <tr>
            <td>4</td>
            <td>person-1</td>
            <td>3</td>
            <td>NameChanged</td>
            <td>{ ...}</td>
        </tr>
        <tr>
            <td>5</td>
            <td>person-3</td>
            <td>1</td>
            <td>Created</td>
            <td>{ ...}</td>
        </tr>
    </tbody>
</table>

Unfortunately there is a problem if two transactions are adding an event concurrently to the Event Store (note, that this is about adding events to different streams. It is not the same as the case mentioned in [Optimistic Locking](#optimistic-locking) where events are added concurrently to the _same_ stream). The first transaction would write an event with the global sequence number 3 and the second transaction with number 4. Now if for some reason the second transaction commits first then the global sequence would be 1, 2, 4. As soon as the other transaction commits, the sequence would be 1, 2, 3, 4. A projection might first see the new event with number 4 and store this as the last sequence number read. So it will miss event 3! It will never read this event because the it will store 4 as the highest event processed.

This problem was [filed as a bug](https://github.com/akka/akka-persistence-jdbc/issues/96) some time ago in the [akka-persistence-jdbc project](https://github.com/akka/akka-persistence-jdbc) (note that the actual problem is of course not akka specific). The solution they chose is basically as follows (I haven't really studied the implementation): If there are gaps in the overall sequence number then the Read Journal waits for a configurable amount of time to see whether the gap is filled or not. If the journal sees 1,2,4 then it will note that there is event number 3 missing. It will wait for a moment to see whether 3 appears.

I think this makes it clear that Greg Young is right, and it is not really trivial to implement your own event store.

# Kafka as an Event Store

I don't really know Kafka. So I'm probably not qualified... Still... here are my thoughts on this:

It's of course possible to store events in Kafka. One problem with Kafka as an Event Store is the fact that it is difficult t   o read a "Stream" of events (in the sense of Event Sourcing) efficiently. You can read more about this in this [presentation from Guido Schmutz](https://de.slideshare.net/gschmutz/kafka-as-an-event-store-is-it-good-enough). However I think that there is one essential point missing in his presentation: Locking. At least the approach with the `expectedVersion` (see [Optimistic Locking](#optimistic-locking)) is not feasible with Kafka as far that I know. There is an [open issue](https://issues.apache.org/jira/browse/KAFKA-2260) which basically requests something similar to an `expectedVersion` in Kafka. Another good blog post I found while writing this post is [this one here](https://medium.com/serialized-io/apache-kafka-is-not-for-event-sourcing-81735c3cf5c) which states basically the same I do.

There is another reason one might not want to use Kafka: (Micro?)Services will depend on a (central?) message broker. If Kafka is not used then probably a RDBMS is. So you depend on an RDBMS... You could of course deploy Kafka per microservice but this is probably more of an overhead than do the same with a database. I'm not sure but to me it does not seem to make much sense.

Don't get me wrong. I'm not saying Kafka is bad. It's just not suited as an Event Store. I think that Kafka is probably a very good choice to publish events that have to be consumed by other Bounded Contexts (read about this [below](#integration-with-other-systems)).

# EventStore as an Event Store

[EventStore](https://eventstore.org/) is an Open Source Event Store Implementation by Greg Young. It was one of the first Event Store implementations I came across. I have no real experience other than playing around a bit with it.

As with Kafka I'm not sure whether it is the right approach to have a single, central Event Store for all your applications. Yes I know... it has "High Availability", "Great Performance" etc. And you could of course also deploy EventStore per microservice. EventStore also provides things like "User defined projections" which of course might be great as long as you are aware of the disadvantages of such dependencies. 

# Akka as Event Store

Currently we are using [Akka Persistence](https://doc.akka.io/docs/akka/current/persistence.html) as an "Event Store". There are various implementations. We are currently using [akka-persistence-jdbc](https://github.com/akka/akka-persistence-jdbc) which uses basically a relational table for events and one for snapshots. The nodes that are running our (clustered) application each write their events directly to the database. The database is per microservice. So there is no central broker (however there are different implementations and [one of them](https://index.scala-lang.org/eventstore/eventstore.akka.persistence/akka-persistence-eventstore/7.0.1?target=_2.13) also allows [Greg Young's EventStore](https://eventstore.org/) to be used.

I feel that this is the right choice. Not because we use Akka. The same could be achieved in a different way. The main advantage in my opinion is the fact that the Event Store is per microservice and we do not depend on a message broker.


# Integration with other Systems

As mentioned before, Event Sourcing Events ar primarily used to reconstitute the state of your system. Of course these events could also be directly "published" and then consumed by other Systems and Bounded Contexts.

Personally I don't like this idea very much. The events are very specific. As soon as these events are "in the wild" you have to care about other consumers and not just yourself. Also regarding security (especially privacy) this may get very complicated. There may be information in your system that nobody else must know. But if you just publish all events then you cannot protect this information (or at least it could get complicated very quickly).

There is still an advantage in using Event Sourcing because it possible to listen to specific events and then publish "Integration Events" that are used for communication with other systems. In a "classic" application that persists the current state of the system such events have to be created apart.

However I would definitely consider the direct consumption of Event Sourcing Events inside the same Bounded Context (as in DDD). This includes Read Models/Projections, Process Managers etc.

# Summary

- Implementing Event Stores is not trivial. Use an existing implementation if possible.
- Whatever Event Store you're using: Make sure it allows you to ensure invariants either with Optimistic Locking or by otherwise ensuring sequential processing of commands per Aggregate.
- I prefer an Event Store per microservice over a central Event Store.
- Consume Event Sourcing Events only inside the same Bounded Context.
- Kafka is not suited as an Event Store but might be great for publish events for consumption by other systems or Bounded Contexts

