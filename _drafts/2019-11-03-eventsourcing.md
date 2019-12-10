---
layout: test_post
title:  "Event Sourcing - An Introduction"
categories: [architecture, design, eventsourcing]
comments: true
---

Does the world need yet another introduction on event sourcing? Well... My primary reason is be able to refer to it in my coming posts where I will dive more into specific topics that I encountered at work when started implementing an Event Sourced system at my company.

# A definition

So what's the actual definition of Event Sourcing? The definition should be given by it's inventor, right? So who's the inventor?

Greg young at least "coined" the term CQRS (according to the [DDD Europe website](https://dddeurope.com/2017/speakers/greg-young/)). I'm not sure whether he also coined the term "Event Sourcing". At least he wrote an important "paper" on the topic [which can be downloaded here](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf). But according to this paper the concept of using events as a storage mechanism is not new:

> Before the general acceptance of the RDBMS as the center of the architecture many systems did not store current state. This was especially  true in high performance, mission critical, and/or highly secure systems. In fact if we look at the inner workings of a RDBMS we will find that most RDBMSs themselves not actually work by managing current state!

So Event Sourcing is nothing new but at least it has gained some popularity in the last few years. A good definition of Event Sourcing comes from Martin Fowler:

> The core idea of event sourcing is that whenever we make a change to the state of a system, we record that state change as an event, and we can confidently rebuild the system state by reprocessing the events at any time in the future. The event store becomes the principal source of truth, and the system state is purely derived from it. [Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html)

Although I think this definition is good I do not like [Fowler's article on Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) because I think his article can cause confusion and for some parts I do not agree with what he is saying. Also note that the article was written in 2005 and it has a disclaimer saying "As such this material is very much in draft form and I won’t be doing any corrections or updates until I’m able to find time to work on it again".


# So how does it work?

Actually it is quite simple: Instead of storing the current state to a database we only store everything that happens in our domain (the domain events) and derive the current state by going through all events.
 
First look at the traditional way with a database. The table to store persons could look as follows: 

{% plantuml %}
class Person {
  int id;
  String firstName
  String lastName
  String street
  String zip
}
{% endplantuml %}

In order to change the last name the old value is overwritten with the new value. 

In an Event Source system there is a domain event "LastNameChanged" that contains the new  value for the last name.
 
{% plantuml %}
@startuml
:PersonCreated;
:LastNameChanged;
@enduml
{% endplantuml %}

So in order to know the current state of the given person we take the first `PersonCreated` and the apply the changes from the `LastNameChanged` event.

The advantage is obvious: We don't loose information. In the traditional application that uses a database we loose forever the person's last name before the last name has been changed. Of course there are solutions for this. A well-known library would be [envers](https://docs.jboss.org/envers/docs/) that "enable easy auditing of persistent classes". 
