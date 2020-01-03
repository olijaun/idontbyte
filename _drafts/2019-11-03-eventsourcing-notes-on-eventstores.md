---
layout: test_post
title:  "Event Sourcing - Notes on EventStores"
categories: [architecture, design, eventsourcing]
comments: true
---

This is the first part in a series about Event Sourcing. In the past year I was involved in the development of a Java application using Event Sourcing. Actually we did it twice using two different approaches. In this post I'd like to share some thoughts about Event Stores.

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).


# What is an Event Store?

In an Event Sources System the events needed to reconstitute the state of the system are stored in an Event Store. There are many different types of events in an architecture landscape and they may serve different purposes. Events might be used to communicate with other bounded contexts. Event Sourcing Events might be used for this. However the primary purpose of Event Sourcing Events is to reconstitute the system's state.

An Event Store must be capable of storing streams of events. An event stream is just an ordered list of events belonging to an entity (an aggregate root in terms of DDD). The events are stored in order they are emitted. An example of an aggregate root might be a "Person". There may be multiple Persons A, B, C. For each of these Persons there is a separate stream. The streams might be called person-A, person-B, etc. The order of the events inside a stream is very important because the events are replayed in order. It would not make any sense to replay the "PersonDeleted" event before the "PersonCreated" event. 

# Optimistic Locking

Vaughn Vernon's [IDDD book](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design) and [Greg Young's papers on CQRS](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) both show a similar interface for an Event Store. Here's Greg Young's Version:

```csharp
public interface IEventStore {
 void SaveChanges(Guid AggregateId, int OriginatingVersion, IEnumerable<Event> events);
 IEnumerable<Event> GetEventsFor(Guid AggregateId);
}
```

The naming implies Domain Driven Design (DDD). However it is not necessary to use DDD but it somehow fits very well. Instead of calling it `AggregateId` it could be called `EventStreamId` or similar. However for me the most notable thing is the `OriginatingVersion`. Greg Young explains, that this is used for optimistic locking. So when saving your aggregate (or stream) you specify the version you expect to be in the Event Store when appending your new version. If someone else has modified the aggregate in the meantime then the Event Store will throw an optimistic locking exception. This mechanism is important in order to ensure business rules or invariants.

Imagine a bank account. It must only possible to withdraw money from the account if there is enough money. If there are 42 swiss francs on the account and two people try to withdraw these 42 francs at almost the same time then only one must succeed. 


{% plantuml %}
actor "User" as U
participant "Service" as S
participant "Repository" as R

U -> S: withdraw(42)
S -> R: getAccount("123")
create   "Account" as A
R -> A: create()
return
S <-- R: a: Account
S -> A: withdraw(42)
return
S -> R: store(a)
return
U <-- S:
{% endplantuml %}


I had some discussions with co-workers who argued that message driven systems should be designed "more tolerant". Imagine a reservation system that might overbook seats (book a seat twice for example). There might be a process that sends a cancellation to the customer in the (hopefully) seldom case that this happens. This is of course possible and in some cases this might be the best way but it does not change the fact that sometimes you have "hard" business rules. In DDD this is one of the motivations to put such a logic inside an aggregate. In the end "the business" will decide whether the rule is really invariant.

## Whats wrong with Optimistic Locking?

The problem is that the event store will always check the version. Depending on how the optimistic locking is implemented this can be a performance issue. But apart from that it can be very inconvenient for the users of our system. Imagine again the Bank Account Aggregate: On withdrawal we have to check that there is enough money. This makes sense. But what about depositing money? Does it matter in which order deposits are made? Probably not. So why bother the user with a `ConcurrencyException` or something similar?

It would be simple to support for example `-1` as a value tu the `OriginatingVersion` which would tell the Event Store that it should not check the version. But then how does the repository know in which case the version matters and in which case not? The Repository would need business knowledge.

A solution proposed in the [IDDD book](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design) is that the aggregate is reloaded in case of a `ConcurrencyException` and the command is applied again. The `ConcurrencyException` is not propagated to the user. On the second attempt there might be again a `ConcurrencyException` (although probably unlikely in the real world). So this can be repeated until the operation either succeeds or fails with an actual business exception.

However this also means that if you want to avoid for example overriding of certain attributes by a user just shortly after another user has modified it, then this has to be implemented explicitly. But this is fine because then it is an actual business rule. An example would be the modification of an email address by two users for the same person more or less concurrently which makes probably no sense. But this is a business rule and not a technical issue to be handled by the infrastructure and should be implemented in the aggregate.

## An alternative with actors

An alternative to optimistic locking is to assure that there is always only one instance of a specific aggregate and that commands for each of this instances are applied sequentially. This can be achieved by using for example Actor Frameworks like Akka or Vlingo. I will go into more details on how this could be implemented in a later post. Actors basically have an inbox where they receive messages (e.g. a Command) and process them one by one. If the system is deployed on multiple nodes then sharding is required. Each shard acts as the single source for a set of aggregates.


# Implementation challenges

Greg young writes:

> Although not a trivial exercise to create a production quality Event Storage the overall concepts behind
  an Event Storage are relatively easy
  
Why shouldn't this be trivial? Well, one thing is getting the optimistic locking right. Often database specific stuff has to be taken into account. You have to understand isolation levels etc. 

Another challenge I came across was when I tried to implement my own read journal for Akka. A Read Journal is basically the "Read Side" of the EventStore. Usually you want to subscribe to events in an Event Store. This could be a subscription to a specific aggregate, aggregate type or to specific event types.

For example in case of a Read Model/Projection events are used to update the current state of some aggregates. The Read Model could be rebuilt from scratch each time the system is started. This could get slow if there are many events in the system. Another approach is to continue listening for events at a certain position after the system has been restarted. Let's say the system was shutdown after having read event number 2567. When restarting the system the projection requests all events after 2567. If the projection subscribes to one stream single stream/aggregate then it is simple. The sequence number after 3 is 4 inside the same stream.

But what happens if the projection listens to events of a certain type that can be emitted by multiple aggregates? The simplest approach to this is to have an auto increment field/column that provides a unique sequence number across all streams/aggregates. So the Projection just has to provide the last processed overall sequence number to the Event Store in order to get all following events. Unfortunately there is a problem with this.

What happens if two transactions are adding an event concurrently to the Event Store (e.g. implemented in a RDBMS). Note, that this is about adding events to different streams. It is not the same as the case mentioned in [Optimistic Locking](#optimistic-locking) where events are added concurrently to the _same_ stream. The first transaction would write an event with the autoincrement number 3 and the second transaction with number 4. Now for some reason the second transaction commits first then the overall sequence would be 1, 2, 4. As soon as the other transaction commits, the sequence would be 1, 2, 3, 4. A projection would first see the new event with number 4 and store this as the last sequence number read. So it will miss event 3!

This problem was [filed as a bug](https://github.com/akka/akka-persistence-jdbc/issues/96) some time ago in the akka-persistence-jdbc project. The solution they chose is basically as follows (I'havent really studied the implementation): If there are gaps in the overall sequence number then the Read Journal waits for a configurable amount of time to see whether the gap is filled or not. 

# Kafka as an Event Store

I don't really know Kafka. So I'm probably not qualified... Still... here are my thoughts on this:

It's of course possible to store events in Kafka. One problem with Kafka as an Event Store is the fact that it is difficult to read a "Stream" of events (in the sense of Event Sourcing) efficiently. You can read more about this in this [presentation from Guido Schmutz](https://de.slideshare.net/gschmutz/kafka-as-an-event-store-is-it-good-enough). However I do not agree on everything he says in the presentation. For example I do not understand the approach he describes on page 35 (but I just looked at the slides and did not hear his speech so maybe I'm missing something). However I think that there is one essential point missing in his presentation: Locking. At least the approach with the `OriginatingVersion` (see [Optimistic Locking](#optimistic-locking)) is not feasible with Kafka as far that I know. There is an [open issue](https://issues.apache.org/jira/browse/KAFKA-2260) which basically requests something similar to an `OriginatingVersion` in Kafka.

There is another reason one might not want to use Kafka: (Micro?)Services will depend on a (central?) message broker. Of course it depends. If Kafka is not used then probably a RDBMS is. So you depend on an RDBMS... You could of course deploy Kafka per Microservice but this is probably more of an overhead than do the same with a database. I'm not sure but to me it does not seem to make much sense.

For our projects we have used Akka Persistence with akka-persistence-jdbc so far.

# Integration with
