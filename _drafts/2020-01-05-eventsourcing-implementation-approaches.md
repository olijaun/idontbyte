---
layout: test_post
title:  "Event Sourcing - Implementation Approaches"
categories: [architecture, design, eventsourcing]
comments: true
---

This is the third part in a series about Event Sourcing. In the past year I was involved in the development of a Java application using Event Sourcing. Actually we did it twice using different approaches. In this post I'd like to share some thoughts about the design of these approaches.

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).

# Introduction

There is not just one way to implement an Event Sourced application. Event Sourced applications can be very different. They can range from Banking to Source Code Management (like Git or Subversion). Different domains have different requirements.

Also Event Sourcing is often mentioned along with Domain Driven Design which is not really a requirement but it fits very well and I for myself have always implemented Event Sourcing in the context of DDD applications.

When we've implemented our first Event Sourced application we've used the "Classic Approach" with an Event Store that we've implemented ourselves. We later decided that implementing an own Event Store is not the best idea and started to use [Akka](https://akka.io/). 

Akka is an Actor Framework. Actors cannot be accessed directly. It is only possible to send messages to them. This made it necessary for us to switch to the second approach.

In both cases we maintained a [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) which means (among other things) that the domain logic must not depend on a framework (Akka in our case). I will write another post on how this can be achieved. This post will only focus on the two approaches on a higher level.

# Approach I: The Classic Approach

The [IDDD book](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design) describes what I call the "Classic Approach" with DDD. The classic approach with DDD works more or less like this (this is my own variation and does not reflect 100% what is shown in the iDDD book):

{% plantuml %}
actor "User" as U
participant "Application Service" as AS
participant "Repository" as R
participant "EventStore" as ES
U -> AS: **1** withdraw("X", 42)
AS -> R: **2** getAccount("X")
R -> ES: **3** loadEventStream("X")
R <-- ES: events : List<Events>
create "Account" as A
R ->  A: **4** create(events)
A -> A: **5** apply(events)
R <-- A
AS <-- R: a: Account
AS -> A: **6** withdraw(42)
A -> A: **7** emit MoneyWithdrawnEvent
AS <-- A 
AS -> R: **8** save(a)
R -> A: **9** getEmittedEvents()
R <-- A: emittedEvents: List<Event>
destroy A
R -> ES: **10** append("X", emittedEvents)
R <-- ES
AS <-- R
U <-- AS
{% endplantuml %} 

1. The user invokes the withdraw operation
2. The Application Service uses the Repository to load the Account
3. In order to load the account the Repository reads the stream of past events of the given account from the EventStore.
4. It then creates an instance of the Account Aggregate and passes in the stream of events
5. The Account Aggregate applies all events in the stream in order to reconstitute its current state
6. Now the Application Service can invoke the withdraw-Method on the Account Aggregate
7. The account emits a new event that states that money has been withdrawn
8. The Application Service tells the Repository to save the Account Object
9. The Repository gets all newly emitted events from the Aggregate
10. The Repository appends the events to the appropriate stream in the Event Store

Note:
- It uses a classic DDD approach using a Repository to load Aggregates. The explicit call to `save()` is not compliant with the [original definition by Eric Evans](https://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf) which states that a Repository is "a service that can provide the illusion of an in-memory collection of all objects of that aggregate’s root type". But this problem exists with other implementations as well and is kind of an "accepted violation" of DDD :-)
- The Aggregate is loaded from the Event Store every time a new request is processed. This could be a performance issue. In such cases snapshots may be used (refere to [Greg Young's Paper](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) if you don't konw what snapshots are)
- The fact that there might be multiple concurrent requests makes it necessary to have some kind of locking in order to maintain consistency in the aggregate. Check out [my other post about Event Stores]({% post_url 2020-02-23-eventsourcing-notes-on-eventstores %}) if you want to know more.
- The User's invocation is shown as a synchronous one. It could be asynchronous but still loading the aggregate, withdrawing the money and saving the aggregate occupies one thread.

The Application Service's code could look like this (ignoring error handling etc.):

```java

public class ApplicationService {
    
    private AccountRepository repository;
    
    // ...
    
    public void withdraw(String accountId, int amount) {
        
        Account a = repository.getAccount(accountId);
        a.withdraw(amount);
        repository.save(a);
    }
}

```

# Approach II: The Reactive Approach

In the Reactive Approach you cannot invoke methods directly on an Aggregate. You can only send messages to an actor which goes with the [reactive manifesto](https://www.reactivemanifesto.org/) which states:

> Reactive Systems rely on asynchronous message-passing to establish a boundary between components that ensures loose coupling, isolation and location transparency.

In our system we implemented this approach using [Akka](https://akka.io/) and [Akka Persistence](https://doc.akka.io/docs/akka/current/persistence.html). Akka is an implementation of the [Actor Model](https://en.wikipedia.org/wiki/Actor_model) which was first described in 1973 by Carl Hewitt. Akka is implemented in Scala but has a Java API. A similar implementation could probably achieved by using [vlingo](https://vlingo.io/) which is another implementation of the Actor Model. At the time we looked at it we didn't know vlingo so we went for Akka. There is another framework which is called [Axon Framework](https://axoniq.io/) and it does use a reactive approach as well.

Instead of a Repository a Command Gateway is used. A Command Gateway receives a command and sends it to the aggregate.

{% plantuml %}
actor "User" as U
participant "Application Service" as AS
participant "Command Gateway" as CG
participant "Account Actor" as AC
participant "Account" as A
participant "EventStore" as ES

U -> AS: **1** withdraw("X", 42)
AS -> CG: **2** handle(cmd: WithdrawCmd)
CG -> AC: **3** send(cmd: WithdrawCmd)
alt aggregate not loaded yet
  AC -> ES: **4.1** loadEvents("X")
  AC <-- ES: events: List<Events>
  create A
  AC -> A: create()
  return
  AC -> A: **4.2** apply(events)
  return
end
AC -> A: **5** handle(cmd: WithdrawCmd) : List<Events>
AC <-- A:
AC -> ES: **6** save(events)
return
...
CG <- AC: **7** send(r: Response)
{% endplantuml %}

1. The user invokes the withdraw operation
2. The Application Services creates an appropriate command message and invokes the Command Gateway
3. The Command Gateway sends the command message to the Actor (this is a potential remote invocation)
4. The (persistent) Actor receives the command message. If the actor does not exist yet in memory then it will be loaded using the persisted events from the Event Store. How this happens exactly is left to the Actor Framework. The important thing here is to note that it is not necessarily reloaded on each request. When the Actor's Events are replayed it also forwards them to the actual Aggregate it represents. The Aggregate could actually be implemented directly in the Actor class but we decided to separate this (Clean Architecture).
5. The Actor invokes the withdraw operation on the Aggregate. The Aggregate returns the events which are a result of this operation.
6. The Actor persists these returned events using the Event Store 
7. At some point the Actor will send a confirmation that the command has been processed

Note:

- The invocation is asynchronous but there is a response for the command eventually. How a client can correlate such a response with his original request is left open here. A polling or notification mechanism can be used for this purpose.
- There is only one actor for each aggregate (a specific instance like Account "X"). There may be multiple actors for each aggregate type.
- No matter how many commands are sent to the actor "X": The Actor "X" will always only process one message at a time. No additional locking is required to ensure the aggregate's invariants.
- In order to achieve this in a deployment with multiple nodes something like "sharding" or "partitioning" is required where each shard acts as the single source for a set of aggregates.

The Application Service's code could look like this (ignoring error handling etc.):

```java

public class ApplicationService {
    
    private AccountCommandGateway gateway;
    
    // ...
    
    public void withdraw(String accountId, int amount) {
        WithrawCommand command = new WithdrawCommand(accountId, amount);
        gateway.withdraw(command);
    }
}

```

What's interesting is that this is actually shorter than the Classic Approach. No explicit save is required in the ApplicationService.

# Comparison

There is significant interest today in Reactive Systems and reactive Programming. I'm not qualified to make any statements about how they perform etc. At least from theoretical standpoint they may perform better -- at least in certain cases and depending on the definition of "performance". Messaging etc. are definitely an overhead when compared to direct method invocations. But as soon as you need scalability and high throughput the Reactive Approach gets more interesting. 

From a programmers standpoint the classic approach looks probably more familiar. The main work was to implement the basic framework we build around Akka in order to achieve a Clean Architecture. For me it was difficult to get into the message driven and asynchronous thinking. Also testing of such systems is not simple and you have to get accustomed first. Now that we have this framework we can focus on business features. Also as you can see in the code snippet some thing even get simpler.


