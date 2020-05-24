---
layout: test_post
title:  "Event Sourcing - Thinking Differently"
categories: [architecture, design, eventsourcing]
comments: true
---

This is the fourth part in a series about Event Sourcing. In the past year I was involved in the development of a Java application using Event Sourcing. Actually we did it twice using two different approaches. In this post I'd like to share some thoughts about commands in the context of CQRS and Event Sourcing. See also my other post on Event Sourcing:

{% include_relative event_sourcing_series.md %}

This post assumes that you know what Event Sourcing is. If not then I recommend that you read [this Document from Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf).

# Keep Read and Write Model separate when using CQRS

As I mentioned earlier: The basic definition of Event Sourcing is that you reconstitute the state of the application by replaying all past events. Event Sourcing does not necessarily mean that CQRS needs to be used although often it might be very useful. CQRS splits the application in two parts: the Read Model and the Write Model. This allows for better scaling and separation of concern. If CQRS is used it is important to really keep the two sides separate. 

You might ask: "so what, where's the problem"? The problem is, that it might be necessary to rethink the application's design in order to achieve this. Let's look at an example.

Assume a system that manages memberships. Each membership states the year and references a member. The RESTful service might look like this (Note that for the sake of simplicity I do not use [HATEOS](https://en.wikipedia.org/wiki/HATEOAS) and other fancy stuff):

```
POST /memberships

{
    "year": 2020,
    "memberId": "123"
}
``` 

Creating and saving a membership could look as follows. Note that creating a `Membership` automatically generates an ID for us:

```java
public class MembershipApplication {
    public MembershipId createMembership() {

        Membership m = new Membership(memberId, year);
        repository.save(m);
        return m.getId();
    }
}
```

Now the business defines a business rule which states that there can only be one membership per member and year. In a "normal" DDD application one could do the following:

```java
public class MembershipApplication {
    public MembershipId createMembership(MemberId memberId, int year) throws AlreadyMemberForGivenYear  {

        Collection<Membership> memberships = repository.find(memberId, year);

        if(!memberships.isEmpty()) {
            throw new AlreadyMemberForGivenYear();
        }

        Membership m = new Membership(memberId, year);
        repository.save(m);
        return m.getId();
    }
}
```

Well ok, this check would probably belong in a domain service and not an application service. But that is not the point. Also to actually guarantee that there won't be two memberships for the same member and year this would not be sufficient in a classic DDD application neither. In a concurrency situation the `isEmpty()` could be OK but then another thread saves the membership just after the check. This could be fixed with a unique constraint on the database. But then we are actually implementing a business rule in the infrastructure.

When using CQRS the problem becomes even more obvious: You cannot make a unique constraint on an Event Store for this check. Sure, there are usually some unique constraints in an Event Store. E.g. the event ID or the sequence numbers. But they are not specific for a business requirement. Also in CQRS you have to assume that the store will be up-to-date probably hours after an aggregate has been modified. Other reasons to use CQRS are scalability and availability. By using the read model each time a new membership gets created there is load generated on the Read Model. Further the Write Model's performance and availability depends on the Read Model which is not desirable.

One solution is using a `MembershipId` that combines `memberId` and `year`:

```java
public class MembershipId {
    private final MemberId memberId;
    private final int year;
    // ...
    public String toString() {
        return memberId.toString() + "-" + year;
    }
}
```

So the `MembershipId` used to store the aggregate might be `123-2020`. In order to check if a membership for the given member exists one can just try to load it:

```java
public class MembershipApplication {
    public MembershipId createMembership(MemberId memberId, int year) throws AlreadyMemberForGivenYear  {

        Membership m = new Membership(memberId, year);
        Membership memberships = repository.get(membership.getId());
        if(memberships != null) {
            throw new AlreadyMemberForGivenYear();
        }
        repository.save(m);
        return m.getId();
    }
}
```

It would be necessary to think about `save()` and how it behaves if there exists an aggregate with this ID already. One option would be to make two different methods like `create()` and `update()`. `create()` would throw an exception if an aggregate with the given ID already exists.

There is another disadvantage of using such a combined key: If the business suddenly decides that there may now be multiple memberships for the same member and year then what? This would mean that we have to change the key of all our `Membership` aggregates.

# Thinking differently

I'm not saying the problem here applies only to Event Sourced applications with CQRS. However I think the fact that it is not possible to randomly search the database forces you to think differently and "harder" because you simply cannot do some dirty hacks. This will eventually lead to a better solution as I will explain now:

In the Membership Example a new aggregate could be added: `Member`. This allows to do the following:

```java
public class MembershipApplication {
    public MembershipId createMembership(MemberId memberId, int year) throws AlreadyMemberForGivenYear  {

        Member member = memberRepository.get(memberId);
        if(member == null) {
            member = new Member(memberId);
        }
        Membership membership = new Membership();
        member.addMembership(membership); // may throw AlreadyMemberForGivenYear
        memberRepository.save(member);
        return membership;
    }
}
```

I think this is much better. The business rule can now be fully implemented in the domain. We do not need a combined key that must be changed if the business rule changes in the future.

There is however something to be aware of in the example: I made `Membership` an entity of the `Member` aggregate. When a client creates a new Membership by posting to `/memberships` we want to return the ID of the newly created membership in order to access it afterwards by using GET on `/memberships/{membershipId}`.

It would be possible to still return a combined key in the RESTfull service. So we could return again something like `123-2020`. Using this key it is possible to extract the member's ID, load the `Member` aggregate and then locate the membership using the year in order to update then the membership. However image again that the business rule changes, and it is now possible to have multiple memberships for the same member and year. The ID would change. Any system that is holding a reference to the membership (for example the accounting system) will break.

this can fixed by generating something like a UUID for each membership. When posting to `/memberships` the service would return for example `e06d8423-f90c-4c59-839e-e71ade68f37f`. Reading is not a problem, because the Read Model can contain this ID and it is possible to simply look it up. But now there is a problem when updating a membership: Updating is a write operation so it is only allowed to use the Write Model. But how can we access the `Membership` if we don't know the Member's ID?

A simple solution is to still use a combined ID but this time a combination of the `MemberId` and the generated `MembershipId`. E.g. `123@e06d8423-f90c-4c59-839e-e71ade68f37f` (I'm using '@' here as a separator because the UUID already contains dashes). This ID is stable and allows to change the business rules for adding Memberships without affecting the `MembershipId`

If you're company insists in using a specific format for IDs then you would have to do some kind of translation from your combined ID to one that satisfies your companies rules.

# Towards Self Contained Systems

A Self Contained System (SCS) has the following characteristics (among others, see [here](https://scs-architecture.org/));

- Each SCS is an autonomous web application. [...]
- Communication with other SCSs or 3rd party systems is asynchronous wherever possible. [...]

Let's look again at our membership service:

```java
 public class MembershipApplication {
     public MembershipId createMembership(MemberId memberId, int year) throws AlreadyMemberForGivenYear  {
 
         Member member = memberRepository.get(memberId);
         if(member == null) {
             member = new Member(memberId);
         }
         MembershipId membershipId = member.addMembership(memberId, year); // may throw AlreadyMemberForGivenYear
         memberRepository.save(member);
         return membershipId;
     }
 }
 ```
The service first checks if a Member with the given ID exists. If not, it creates a new one. This is one possibility. The problem is that a client might provide a non-existent memberId. The service could check whether the Member actually exists by calling the member system using a RESTfull service. The example uses the `MemberPort` for this (I use the term port here to emphasize the Ports and Adapter Architecture, aka Hexagonal).

```java
 public class MembershipApplication {
     public MembershipId createMembership(MemberId memberId, int year) throws AlreadyMemberForGivenYear, MemberNotFoundException  {
 
         Member member = memberRepository.get(memberId);
         if(member == null) {
             if(memberPort.memberExists(memberId)) {
                 member = new Member(memberId);
             } else {
                 throw new MemberNotFoundException();
             }
         }
         MembershipId membershipId = member.addMembership(memberId, year); // may throw AlreadyMemberForGivenYear
         memberRepository.save(member);
         return membershipId;
     }
 }
 ```
This only replicates the member if it exists. This is fine, but it is not really self contained. We should communicate asynchronously with other systems. So the solution would be to subscribe to member events and replicate members when receiving events. This way the system is very independent, and it is possible to add memberships event if the member system itself is not available.

Here's a sketch of how this could look like:

```java
 public class MembershipApplication {

     public MembershipId createMembership(MemberId memberId, int year) throws AlreadyMemberForGivenYear, MemberNotFoundException  {
 
         Member member = memberRepository.get(memberId);
         if(member == null) {
             throw new MemberNotFoundException();
         }
         MembershipId membershipId = member.addMembership(memberId, year); // may throw AlreadyMemberForGivenYear
         memberRepository.save(member);
         return membershipId;
     }
     
     // handle events from member system and keep members up-to-date
     private onEvent(MemberEvent memberEvent) {
         // [...]
         memberRepository.save(member);
     }
 }
 ```

How you process exactly the member events depends of course on your messaging system and framework. This is just a sketch. It is however important to note that updating the members is done in the Write Side of your application. It is not to be confused with the event processing that it probably going on elsewhere in this application in order to update the Read Model.

# Summary

When implementing an event-sourced application with CQRS for the first time the fact that you must not access the Read Model when writing may be unfamiliar. However I think it will lead to a better design in respect to DDD eventually. This is of course my personal and highly subjective opinion.