---
layout: test_post
title:  "Event Sourcing - Commands can be rejected"
categories: [architecture, design, eventsourcing]
comments: true
---

This is the second part in a series about Event Sourcing. In the past year I was involved in the development of a Java application using Event Sourcing. Actually we did it twice using two different approaches. In this post I'd like to share some thoughts about commands in the context of CQRS and Event Sourcing. See also my other post on Event Sourcing:

{% include_relative event_sourcing_series.md %} 

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).

# Difference between CQS and CQRS

Bertrand Meyer described the Command Query Segregation Pattern (CQS): "It states that every method should either be a command that performs an action, or a query that returns data to the caller, but not both" (see [Wikipedia](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)).

```java

// mutates state but does not return anything
public void command(int input) {
    this.state = input;
}

// returns a value but does not mutate state
public int query() {
    return state;
}
```

So what is CQRS then? In Command Query Responsibility Segregation (CQRS) the application is split into a query/read and a command/write side which allows for better scaling and separation of concern. Usually reading is performed more often than writing. The query side can be optimized for reading and even be deployed independently of the command side (note that the arrows in the diagram show dependencies not data flow).

{% plantuml %}
database "Event Store" as ES
[UI] as UI


component "Application" as A {
  [Command Side] as C
  [Query Side] as Q
}

UI -down-> C : send command
UI -down-> Q : query data

C -down-> ES : store events 
Q -down-> ES : read events

{% endplantuml %}

In eventsourced applications CQRS can make a lot of sense. With just Event Sourcing it can be difficult and/or slow to perform e.g. queries on your event streams. With CQRS the query side builds so called Projections (or Read/Query Models) using the events produced by the command side. This is usually done asynchronously. One important consequence of this is that the query side might not represent the current state. For example a UI that writes some data and reads them shortly afterwards might not read what it has just written. This has to be taken into account. The advantages are separation of concern and performance among others.

Be aware of the fact that the command side must not access the query side! Note that there is no such arrow in the diagram.

# Commands can fail or be rejected

It may seem trivial but it's important to note that a command in CQRS may fail or it might be rejected. Greg Young's [CQRS Example Application](https://github.com/gregoryyoung/m-r/tree/master/SimpleCQRS) does clearly show this: 

```csharp
public class InventoryItem : AggregateRoot {

    // ...

    public void ChangeName(string newName)
    {
        if (string.IsNullOrEmpty(newName)) throw new ArgumentException("newName");
        ApplyChange(new InventoryItemRenamed(_id, newName));
    }
    
    // ...
}
```

The aggregate root `InventoryItem` simply throws an `ArgumentException` if `newName` is null. No event is persisted in this case.

An event in contrast states a fact, something that happened. An event cannot be rejected. 

Let's look at an example where a command is processed and finally an event is emitted: If a user withdraws 10 CHF from his account which has a balance of 42 CHF then everything is fine. An event will be emitted that states that 10 CHF have been withdrawn. Note that the following sequence diagrams are very schematic and do not show UIs etc.

{% plantuml %}
actor "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U -> S: withdraw 10 CHF
S -> ES: load account
S <-- ES
S -> S: check balance
S -> ES: append "Withdrawn" Event
S <-- ES:
U <-- S: ok message
{% endplantuml %}

If the user tries to withdraw 50 CHF from his account that has a balance of 42 CHF then the command to do so must be rejected. There is no event stored in the Event Store. Of course business people might still decide that they want such attempts to be stored but usually this is nothing to be stored in an Event Store. It could be stored in a log file or a special table etc. 

{% plantuml %}
actor "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U -> S: withdraw 50 CHF
S -> ES: load account
S <-- ES
S -> S: check balance
U <-- S: error message
{% endplantuml %}

My reasoning is that in eventsourcing events are primarily used to reconstitute the state of the system. A rejected command is not needed to reconstitute state.

In Microsoft's [a CQRS Journey](http://cqrsjourney.github.io/) there is an example involving a seat reservation: A user "registers to a conference" via an "Order" Aggregate. The Order Aggregate emitts an "OrderPlaced" Event which is then acted upon by the Process Manager which calls the "ConferenceSeatsAvailabilityAggregate" to make a seat reservation. If the reservation succeeds a "ReservationAccepted" Event is emitted. If it fails a "ResevationRejected" Event is emitted. I don't really see why it is necessary to use events for this. The command could just be rejected. In a later stage they change the example to put the seats that could not be reserved to a waiting list. OK, that could be a reason but then this could also be done by the ProcessManager when he gets the outcome of the command. And then one could also argue that the outcome of a command is also a kind of event. This is a valid argument but I still think it is not necessary to persist this kind of "command-events".

Commands may be executed synchronous or asynchronously. This does not change the fact that they can be rejected of course. In case of asynchronous command execution It is necessary to have a means to get a response for a command. The caller might want to know the reason for the rejection in order to display it in a UI etc. This could be done by polling for the state of command execution (a more elegant solution might be to use to user technologies like [Server Sent Events](https://en.wikipedia.org/wiki/Server-sent_events) etc. in order to avoid polling). As in synchronous execution there will probably a timeout for polling the state. One solution in such cases is to execute the command again which could lead to problems if the system called is not idempotent ([see my post on Idempotence]({% post_url 2019-09-23-Idempotence %}))

{% plantuml %}
actor "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U ->> S: withdraw 50 CHF
S -> ES: load account
S <-- ES
S -> S: check balance
loop for 1 minute
  U -> S: poll for command state
  U <-- S:
end
{% endplantuml %}



# Summary

Regardless of whether commands are sent synchronously or asynchronously to the write side of a CQRS application they may be rejected or they may fail. The caller has to react appropriately.