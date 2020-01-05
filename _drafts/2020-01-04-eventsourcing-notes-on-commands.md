---
layout: test_post
title:  "Event Sourcing - Notes on commands"
categories: [architecture, design, eventsourcing]
comments: true
---

This is the second part in a series about Event Sourcing. In the past year I was involved in the development of a Java application using Event Sourcing. Actually we did it twice using two different approaches. In this post I'd like to share some thoughts about commands in the context of CQRS and Event Sourcing.

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).

# CQS and CQRS

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

In Command Query Responsibility Segregation (CQRS) the application is split into a query/read and a command/write side which allows for better scaling and separation of concern. Usually reading is performed more often than writing. The read side can be optimized for reading and even be deployed independently of the write side.

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

The aggregate root simply throws an `ArgumentException` if `newName` is null. No event is persisted in this case.

An event in contrast states a fact, something that happened. It is note possible to "reject" the event. If a user withdraws 10 CHF from his account which has a balance of 42 CHF then everything is fine. An event will be emitted that states that 10 CHF have been withdrawn. Note that the following sequence diagrams are very schematic and do not show UIs etc.

{% plantuml %}
actor "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U -> S: withdraw 10 CHF
S -> S: check balance
S -> ES: store "Withdrawn" Event
S <-- ES:
U <-- S: ok message
{% endplantuml %}

If the user tries to withdraw 50 CHF from his account that has a balance of 42 CHF then the command to do so will be rejected. There will not be an event stored in the Event Store. Of course the business might still decide that they want such attempts to be stored but usually this is nothing to be stored in an Event Store. It could be stored in a log file. In Microsoft's [a CQRS Journey](http://cqrsjourney.github.io/) there is an example involving a seat reservation: A user "registers to a conference" via an "Order" Aggregate. The Order Aggregate emitts an "OrderPlaced" Event which is then acted upon by the Process Manager which calls the "ConferenceSeatsAvailabilityAggregate" to make a seat reservation. If the reservation succeeds a "ReservationAccepted" Event is emitted. If it fails a "ResevationRejected" Event is emitted. I don't really see why this is necessary. In a later stage they change the example to put the seats that could not be reserved to a waiting list. OK, that could be a reason but then this could also be done by the ProcessManager when he gets the outcome of the command. And then one could also argue that the outcome of a command is also a kind of event. This is a valid argument but I think it is not necessary to persist this kind of events.

{% plantuml %}
actor "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U -> S: withdraw 50 CHF
S -> S: check balance
U <-- S: error message
{% endplantuml %}

A synchronous call directly returns the response although in case of a timeout it is not possible to know about the outcome.

In case of asynchronous command execution It is necessary to have a means to get a response for a command. The caller might want to know the reason for the rejection in order to display it in a UI etc. This could be done by polling for the state of command execution. As in synchronous execution there will probably a timeout for polling the state. One solution is such cases is to execute the command again which could lead to problems if the system called is not idempotent ([see my post on Idempotence]({% post_url 2019-09-23-Idempotence %}))

{% plantuml %}
actor "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U ->> S: withdraw 50 CHF
S -> S: check balance
loop for 1 minute
  U -> S: poll for command state
  U <-- S:
end
{% endplantuml %}

A more elegant solution might be to use to user technologies like [Server Sent Events](https://en.wikipedia.org/wiki/Server-sent_events) etc. in order to avoid polling.

# Summary

Regardless of whether commands are sent synchronously or asynchronously to the write side of a CQRS application they may be rejected or they may fail. The caller has to react appropriately.