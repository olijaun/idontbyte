---
layout: test_post
title:  "Event Sourcing - Clean Architecture"
categories: [architecture, design, eventsourcing]
comments: true
---

This is the fourth part in a series about Event Sourcing. In the past year I was involved in the development of a Java application using Event Sourcing. Actually we did it twice using different approaches. In this post I'd like to share some thoughts about Event Stores. See also my other post on Event Sourcing:

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).


# What is Clean Architecture?

I really enjoyed Robert C. Martin's Book on Clean Architecture. Also his [blog post](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) on the same topic is quite interesting. Hexagonal Architecture, Onion Architecture etc. are all similar: Instead of having a layered architecture where each layer must depend only on the layer below they propose an architecture where the domain model is in the center and the UI, Database etc. depend on the domain model and not vice versa.

![My helpful screenshot](/assets/clean_arch.png)

Robert C. Martin says:

> The outermost layer is generally composed of frameworks and tools such as the Database, the Web Framework, etc. [...] This layer is where all the details go. The Web is a detail. The database is a detail. We keep these things on the outside where they can do little harm.

Let's make an example comparing it to the Layered Architecture. Remember [Data Access Objects (DAOs)](https://en.wikipedia.org/wiki/Data_access_object)? 

{% plantuml %}
hide empty members

package org.jaun.app.domain {
  class MyBusinessLogic
  
  MyBusinessLogic --> org.jaun.app.infrastructure.MyDao
}

package org.jaun.app.infrastructure {
  interface MyDao
  class MyJdbDaoImpl
  
  MyDao <|-- MyJdbDaoImpl
}
{% endplantuml %}

So here the Business Logic depends on the infrastructure (the lower layer). With Clean Architecture this changes to:

{% plantuml %}
hide empty members

package org.jaun.app.domain {
  class MyBusinessLogic
  interface org.jaun.app.domain.MyRepository
  
  MyBusinessLogic --> org.jaun.app.domain.MyRepository
}

package org.jaun.app.infrastructure {
  
  class MyJdbcRepositoryImpl
  
  org.jaun.app.domain.MyRepository <|-- MyJdbcRepositoryImpl
}
{% endplantuml %}
  
Note that I changed DAO to Repository to emphasize that the Repository is nothing "technical". However this is not the main point. The main point is that the Interface `MyRepository` is now in the `org.jaun.app.buisiness` package and the infrastructure implements it.


# Clean Architecture for Event Sourcing

Clean Architecture means that frameworks like Spring, JPA, etc. are a detail. You should not use them in your domain. You can use them in your infrastructure code however. Some people won't like this idea. It means that you cannot use all your fancy `@Autowired` annotations etc. However it does not mean that you cannot use [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection). Dependency injection does not require annotations!

In respect to Event Sourcing this means that you don't want a dependency on the framework you're using for Event Sourcing from your domain model.

Let's have a look at [Axon](https://axoniq.io/) for example. Axon provides a framework for eventsourced applications. Here's an example aggregate (taken from [their web page](https://docs.axoniq.io/reference-guide/implementing-domain-logic/command-handling/aggregate)): 


```java
import org.axonframework.commandhandling.CommandHandler;
import org.axonframework.eventsourcing.EventSourcingHandler;
import org.axonframework.modelling.command.AggregateIdentifier;

import static org.axonframework.modelling.command.AggregateLifecycle.apply;

public class GiftCard {

    @AggregateIdentifier // 1.
    private String id;

    @CommandHandler // 2.
    public GiftCard(IssueCardCommand cmd) {
        // 3.
       apply(new CardIssuedEvent(cmd.getCardId(), cmd.getAmount()));
    }

    @EventSourcingHandler // 4.
    public void on(CardIssuedEvent evt) {
        id = evt.getCardId();
    }

    // 5.
    protected GiftCard() {
    }
    // omitted command handlers and event sourcing handlers
}
```

As you can see this depends on Axon Annotations in order to work. The aggregate is part of the domain model. It should not depend on a framework. The framework should be a detail. It should be possible to replace Axon with something else without touching the domain model.

We have used [Akka](https://akka.io) for Event Sourcing as I mentioned in previous articles. Is Akka better than the Axon Framework regarding Clean Architecture? That's probably not the point. The Axon Framework has a specific way to implement an Event Sourced application. It's probably more suitable to compare Axon with [Lagom](https://www.lagomframework.com/) (which is based on Akka). So akka-persistence is more low-level than Axon or Lagom and it's probably simpler to adapt it to your needs and keep the domain clean.

However even with frameworks like Axon it would be possible to achieve a Clean Architecture. Let's refactor the Axon example from above:

```java
package org.giftcard.infrastructure;

import org.giftcard.domain.GiftCard;

import org.axonframework.commandhandling.CommandHandler;
import org.axonframework.eventsourcing.EventSourcingHandler;
import org.axonframework.modelling.command.AggregateIdentifier;

import static org.axonframework.modelling.command.AggregateLifecycle.apply;

public class AxonGiftCard {

    private GiftCard giftCard;

    @AggregateIdentifier // 1.
    private String id;

    @CommandHandler // 2.
    public AxonGiftCard(IssueCardCommand cmd) {
        // 3.
        giftCard = new GiftCard(cmd);
        List<Event> events = giftCard.getEmittedEvents();
        events.forEach(event -> 
            apply(new CardIssuedEvent(cmd.getCardId(), cmd.getAmount())));
        events.clearEmittedEvents();
    }

    @EventSourcingHandler // 4.
    public void on(CardIssuedEvent evt) {
        id = evt.getCardId();
        giftCard.on(evt);
    }

    // 5.
    protected GiftCard() {
    }
    // omitted command handlers and event sourcing handlers
}
```

Don't get me wrong. I'm not saying that this is a great solution. But it shows how you can move the framework into the infrastructure (note the package declaration I've added). So I renamed the class to `AxonGiftCard` and I'm basically delegating everything to the newly created `GiftCard` which is a framework independent Aggregate Root (residing in the package `org.giftcard.domain`). Due to the fact that I have never used Axon I'm not sure what to do with the id. I assume it is required, so I have the id in `AxonGiftCard` and `GiftCard`.

The problem with this refactoring is that it basically renders everything useless that Axon proponents probably like about Axon. So if you insist in a clean architecture then Axon is probably getting in the way too much.