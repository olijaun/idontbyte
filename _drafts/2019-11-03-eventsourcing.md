---
layout: test_post
title:  "Event Sourcing - Implementation Considerations"
categories: [architecture, design, eventsourcing]
comments: true
---

In the last month we have implemented a Java application using Event Sourcing. Actually we did it twice using two different designs. In this post I'd like to show the two approaches and their advantages and disadvantages. 

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).


# Why Event Sourcing?

So why did we actually choose Event Sourcing for our system? To be honest: It was not really necessary. Our main goal was to learn about Event Sourcing. The system we've implemented was quite simple and we could have implemented as a simple CRUD application using a relational database. But because it is simple it also served us very well in order to learn something new without failing on a large scale -- if we should fail.

Some of the reasons to use Event Sourcing can be performance and scalability. Especially if you combine it with CQRS the separation of the "read" from the "write" side can be an advantage in this respect. However performance and scalability is not really a concern for our application. 

One business advantage is that you have a complete history of all your events. I think this is already a very good reason to use Event Sourcing. Of course there are other ways for implementing a history. My company has done it four decades using their own "history concept". In the last years new applications were developed using Envers which "aims to provide an easy auditing / versioning solution for entity classes" (see [here](https://hibernate.org/orm/envers/)) in combination with JPA. Personally I think that the Event Sourcing approach is very simple. There are no additional tables needed, just the event streams.

There is also another business advantage of using events: Event Sourcing events belong to the domain. The relational database does not. Something like "the customer has moved" is relevant to the business. Something like "The street field in table 'address' was updated to 'xy'" is not. Approaches like EventStorming are getting more popular lately ("a flexible workshop format for collaborative exploration of complex business domains" using events [see here](https://www.eventstorming.com/)).

A technical advantage is that you have events "for free". Nowadays "Event Driven Architectures" are en vogue. Finally they became "main stream". Many applications at my company now publish events in addition to saving their state to the database so that other applications can consume them. Also it seems simple to publish events, which is usually not that trivial in classic applications. One possibility is to send them via a message broker like [Kafka](https://kafka.apache.org/). But then you have two transactions: One writing the current state to the applications database one one writing the event(s) belonging to this state change to the message broker. This can either be solved by using distributed transactions (which nowadays is usually frowned upon) or you poll the database for changes and publish them. With Event Sourcing you only have to care about events. You just store the events and that's it. Imagine in a "classic" application there is a bug and the state has been written correctly to the database but the events published were wrong. What do you do? It might be difficult to fix this situation and inform all the systems that have consumed your wrong events. With Event Sourcing the events might be wrong too. But then also the state of the system is wrong. It would be necessary to create correction events but I think this is simpler to achieve and more clear because the state of the system is always based on the same events that you publish. Just to be clear: I actually do not recommend to publish "Event Sourcing" events directly to other applications outside of your bounded context. But this is another topic.

# Some Notes about Event Stores

After having read the Event Sourcing chapter of Vaughn Vernon's [IDDD book](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design) and [Greg Young's papers on CQRS](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) it was clear to us that we needed an Event Store. Greg Young shows the following example for an Event Store interface:

```csharp
public interface IEventStore {
 void SaveChanges(Guid AggregateId, int OriginatingVersion, IEnumerable<Event> events);
 IEnumerable<Event> GetEventsFor(Guid AggregateId);
}
```

The naming implies Domain Driven Design (DDD) already. However it is not necessary to use DDD but it somehow fits very well. Instead of calling it `AggregateId` it could be called `EventStreamId` or similar. However for me the most notable thing is the `OriginatingVersion`. Greg Young explains, that this is used for optimistic locking. So when saving your aggregate (or stream) you specify the version you expect to be in the Event Store when appending your new version. If someone else has modified the aggregate in the meantime then the Event Store will throw an optimistic locking exception. This mechanism is important in order to ensure business rules or invariants.

Imagine a bank account. It must only possible to withdraw money from the account if there is enough money. If there are 42 swiss francs on the account and two people try to withdraw these 42 francs at almost the same time then only one must succeed.

I had quite some discussions with some people about this who argued that message driven systems should be designed so that they are more tolerant. Imagine a reservation system that might overbook seats (book a seat twice for example). There might be a process that sends a cancellation to the customer in the (hopefully) seldom case that this happens. This is of course possible and in some cases this might be the best way but it does not change the fact that sometimes you have "hard" business rules. In DDD this is one of the reasons to put such a logic inside an aggregate root.

It is however important to distinguish between "technical locking" and "business locking". The technical locking exists to ensure that there are not suddenly two different events with the same sequence number (for the same aggregate). Assume that there is a "Person" aggregate and two people are trying to modify the name concurrently. You could of course just use the technical optimistic locking. This would mean that the guy who presses the "save button" latest will get a concurrency exception. However there might be nicer solutions to this problem: One is to still accept his change without throwing an exception and override the first change. This would mean that the application has to re-read the events and set the `OriginatingVersion` accordingly in order to save the change. Another solution would be to check whether there is actually a difference between the two concurrent changes. Maybe both users are trying to set the same name for the person. So why bother the second user with a concurrency exception?

The second approach we used for event sourcing does not have such an `OriginatingVersion` and still can ensure invariants.

## Technical challenges

Greg young writes:

> Although not a trivial exercise to create a production quality Event Storage the overall concepts behind
  an Event Storage are relatively easy
  
When I read this the first time I thought why shouldn't this be trivial? Well, one thing is proper locking and get the optimistic locking right. Often database specific stuff has to be taken into account for this. You have to understand isolation levels etc. The other thing is that you need to get a sequence number that properly increases 1 by 1.  


# Approach I: The Classic Approach

The [IDDD book](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design) describes what I call the "Classic Approach" with DDD. The classic approach with DDD works more or less like this:

{% plantuml %}
participant "User" as U
participant "Application Service" as AS
participant "Repository" as R
participant "Account X" as A
participant "EventStore" as ES

U -> AS: withdraw
AS -> R: getAccount(X)
R -> ES: load event stream for X
R <-- ES: EventStream: X
R ->  A: create
R <-- A
R -> A: apply events from EventStream X
R <-- A
AS <-- R: Account: X
AS -> A: withdraw()
A -> A: emit MoneyWithdrawnEvent
AS -> R: save(A)
R -> A: getEmittedEvents() : events
R -> ES: append(events)
R <-- ES
AS <-- R
U <-- AS
{% endplantuml %} 

1. The user invokes the withdraw operation
2. The Application Service uses the Repository to load the Account
3. In order to load the account the Repository reads all past events of the given account from the EventStore as an event stream.
4. It then applies all these events to the Account in order to obtain the current state of the account.
5. Now the Application Service can invoke the withdraw-Method on the reconstituted account
6. The account emits a new event that states that money has been withdrawn
7. The Application Service tells the Repository to store the Account Object
8. The Repository stores the Account Object by appending the newly emitted events to the Event Store


# Approach II: The Reactive Approach

In the classic approach we make use of the Repository pattern in order to load and save Aggregates. The call to `withdraw()` is a synchronous and local method call. On each invocation of a business method the Aggregate is reconstituted by applying all past events (this can be accelerated by using snapshosts). 

In the Reactive Approach you cannot invoke methods directly on an Aggregate. You can only send messages to an actor which goes with the [reactive manifesto](https://www.reactivemanifesto.org/) which states:

> Reactive Systems rely on asynchronous message-passing to establish a boundary between components that ensures loose coupling, isolation and location transparency.

In our system we implemented this approach using [Akka](https://akka.io/) and [Akka Persistence](https://doc.akka.io/docs/akka/current/persistence.html). Akka is an implementation of the Actor Model which was first described in 1973 by Carl Hewitt. Akka is implemented in Scala but has a Java API. A similar implementation could probably achieved by using [vlingo](https://vlingo.io/) which is another implementation of the Actor Model. At the time we looked at it Akka was more mature but certainly this could have changed in the meantime. Also the [Axon Framework](https://axoniq.io/) does use the reactive approach. Note that Akka and Vlingo are generic frameworks for reactive applications while Axon is more specific.

Instead of a Repository so called Command Handlers are used. A Command Handler receives a command and sends it to the aggregate.

{% plantuml %}
participant "User" as U
participant "Application Service" as AS
participant "Command Gateway" as CG
participant "Account X Actor" as AC
participant "Account X" as A
participant "EventStore" as ES

U -> AS: withdraw
AS -> CG: handle(cmd: WithdrawCommand)
CG -> AC: send(cmd: WithdrawCommand)
AC -> ES: load event stream for X
AC <-- ES: EventStream: X
AC -> A: apply events from EventStream X
AC <-- A
AC -> A: handle(cmd: WithdrawCommand) : List<Events>
AC <-- A:
AC -> ES: save(events: List<Events>)
CG <-- AC: send(r: Response)
AS <-- CG
U <- AS
{% endplantuml %}

1. The user invokes the withdraw operation
2. The Application Services uses the Command Gateway to send the command to the Actor
3. The Command Gateway sends a command message to the Actor (this may be a remote call)
4. The Actor receives the command
5. The Actor reads all past events for the given aggregate
6. The Actor applies them to the Aggregate
7. The Actor invokes the withdraw  operation
8. The withdraw Operation returns the event(s) that result from performing the withdrawal
9. The Actor stores the events in the event store
10. The Actor sends the response

# Some notes about commands

Bertrand Meyer described the Command Query Segregation Pattern (CQS): "It states that every method should either be  command that performs an action, or a query that returns data to the caller, but not both" (see [Wikipedia](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)).

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

In CQRS the application is split into a query/read and a command/write side which allows for better scaling and separation of concern. Usually reading is performed more often than writing. The read side can be optimized for reading and even be deployed independently of the write side.

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

An event in contrast states a fact, something that happened. It is note possible to "reject" the event. If a user withdraws 10 CHF from his account which has a balance of 42 CHF then everything is fine. An event will be emitted that states that 10 CHF have been withdrawn.

{% plantuml %}
participant "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U -> S: withdraw 10 CHF
S -> ES: store "Withdrawn" Event
S <-- ES:
U <-- S: ok message
{% endplantuml %}

If the user tries to withdraw 50 CHF from his account that has a balance of 42 CHF then the command to do so will be rejected. There will not be an event stored in the Event Store. Of course the business might still decide that they want such attempts to be stored but usually this is nothing to be stored in an Event Store. It could be stored in a log file. In Microsoft's [a CQRS Journey](http://cqrsjourney.github.io/) there is an example involving a seat reservation: A user "registers to a conference" via an "Order" Aggregate. The Order Aggregate emitts an "OrderPlaced" Event which is then acted upon by the Process Manager which calls the "ConferenceSeatsAvailabilityAggregate" to make a seat reservation. If the reservation succeeds a "ReservationAccepted" Event is emitted. If it fails a "ResevationRejected" Event is emitted. I don't really see why this is necessary. In a later stage they change the example to put the seats that could not be reserved to a waiting list. OK, that could be a reason but then this could also be done by the ProcessManager when he gets the outcome of the command. And then one could also argue that the outcome of a command is also a kind of event. This is a valid argument but I think it is not necessary to persist this kind of events.

{% plantuml %}
participant "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U -> S: withdraw 50 CHF
U <-- S: error message
{% endplantuml %}

A synchronous call directly returns the response although in case of a timeout it is not possible to know about the outcome.

In case of asynchronous command execution It is necessary to have a means to get a response for a command. The caller might want to know the reason for the rejection in order to display it in a UI etc. This could be done by polling for the state of command execution. As in synchronous execution there will probably a timeout for polling the state. One solution is such cases is to execute the command again which could lead to problems if the system called is not idempotent ([see my post on Idempotence]({% post_url 2019-09-23-Idempotence %}))

{% plantuml %}
participant "User" as U
participant "Account Service" as S
participant "EventStore" as ES

U ->> S: withdraw 50 CHF
loop for 1 minute
  U -> S: poll for command state
  U <-- S:
end
{% endplantuml %}

A more elegant solution might be to use to user Server Sent Events etc. in order to avoid polling.

