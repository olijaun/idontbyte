---
layout: test_post
title:  "Idempotence"
categories: [architecture, design, webservice]
---

Idempotence is one of most essential properties of a web service. No matter whether it is REST, [SOAP](https://en.wikipedia.org/w/index.php?title=SOAP&oldid=917411406), [gRPC](https://grpc.io/) or any other remoting protocol. However in my experience this aspect is often overlooked or ignored. I am aware that there are many articles about this topic already but often these articles are not tackling the topic in all his aspects. So here is my attempt.


## Definition
An elevator button is usually idempotent. It lights up when it is pressed for the first time. If it is pressed again, it won't change. If it is pressed a hundred times, the elevator will still arrive once.

In mathematics idempotence is defined as follows:

> Idempotence is the property of certain operations in mathematics [...] whereby they can be applied multiple times without changing the result beyond the initial application. ([Wikipedia](https://en.wikipedia.org/w/index.php?title=Idempotence&oldid=917052170))

Here's the definition for "imperative programming':

> in imperative programming, a subroutine with side effects is idempotent if the system state remains the same after one or several calls, in other words if the function from the system state space to itself associated to the subroutine is idempotent in the mathematical sense given in the definition;  ([Wikipedia](https://en.wikipedia.org/w/index.php?title=Idempotence&oldid=917052170))

Here is an example in Java:

{% highlight java %}
{% raw %}
public class MyString {
    
    private String string;
    
    public MyString(String s) {
        string = s;
    }
    
    /** idempotent method */
    public void toUpperCase() {
        string = string.toUpperCase();
    }
    
    /** NON idempotent method */
    public void append(String s) {
        string = string + s;
    }
    
    public String getValue() {
        return string;
    }
}
{% endraw %}
{% endhighlight %}

`toUpperCase()` is idempotent: No matter whether `toUpperCase()` is called once or a hundred times: the result will be the same. `append()` is not idempotent because each time it is being called the string gets longer.


## Distributed Systems

In the context of a single process (as shown above) idempotence is not so exiting... However with distributed systems that are communicating over a network things are getting more interesting.

Firstly it's important to understand why distributed systems are difficult and why there is an important difference between calling a method in the same process and calling a service that lives on a remote system via a network. Years ago Peter Deutsch enumerated eight [Fallacies of Distributed Computing](https://en.wikipedia.org/w/index.php?title=Fallacies_of_distributed_computing&oldid=916589690). The first one was "The network is reliable".
 
 So what can probably go wrong when A and B communicate via a network? A lot. Let's look at a sequence diagram of two services communication. Service A sends a Request to Service B.

{% plantuml %}
participant "Service A" as A
participant "Service B" as B

A -> B: Request
activate B
A <-- B: Response
deactivate A
{% endplantuml %}

If Service A and Service B communicate over a network then this picture is obviously a simplification. So let's draw the network as well:

{% plantuml %}
participant "Service A" as A
participant "Network" as N
participant "Service B" as B

A -> N: Request
activate N
N -> B: Request
activate B
N <-- B: Response
deactivate B
A <-- N: Response
deactivate A
{% endplantuml %}

Also the last image is implying that there is an synchronous communication going on which is not true. The bits and bytes are travelling from A to B and eventually B might create a response that could get lost on its way back to A. In a programming language one may get the illusion that a request is synchronous.

{% highlight bash %}
$ curl -v https://idontbyte.jaun.org
* Rebuilt URL to: https://idontbyte.jaun.org/
*   Trying 185.199.111.153...
* TCP_NODELAY set
* Connected to idontbyte.jaun.org (185.199.111.153) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
{% endhighlight %}

We get the impression that [curl](https://curl.haxx.se/) fetches the page at http://idontbyte.jaun.org synchronously. curl terminates as soon as it receives the response. Under the hood what this call does is send some bytes to the server and then wait to see if it ever gets some bytes as a response. The correlation of request and response is done by the underlying protocol.

What happens if the page is responding very slowly? I'm using [slowwly](http://slowwly.robertomurray.co.uk) here to simulate a slow page. The parameter `--max-time 1` tells curl to wait one second for response:

{% highlight bash %}
$ curl --max-time 1 -v http://slowwly.robertomurray.co.uk/delay/3000/url/http://www.google.ch
*   Trying 34.241.172.109...
* TCP_NODELAY set
* Connected to slowwly.robertomurray.co.uk (34.241.172.109) port 80 (#0)
> GET /delay/3000/url/http://www.google.ch HTTP/1.1
> Host: slowwly.robertomurray.co.uk
> User-Agent: curl/7.58.0
> Accept: */*
> 
* Operation timed out after 1000 milliseconds with 0 bytes received
* stopped the pause stream!
* Closing connection 0
curl: (28) Operation timed out after 1000 milliseconds with 0 bytes received
{% endhighlight %}

curl returns a timeout error in this case. Now look at what can go wrong when A communicates with B:

{% plantuml %}
participant "Service A" as A
participant "Network" as N
participant "Service B" as B
autonumber
A ->x N: Request
activate N
N ->x B: Request
activate B
N x<-- B: Response
deactivate B
A -> A: Timeout

A x<-- N: Response
deactivate A
{% endplantuml %}

1. A cannot connect to the network
2. A can connect to the network but B cannot be reached
3. B cannot connect to the network when it tries to send the response
4. B can send the response to the network but the response never reaches A
5. A is waiting for the response and after a while decides to not wait any longer (timeout)

What can be done if such a problem occurs? One simple solution is to just retry the call. So if the network cannot be reached we just retry to send the data again. If B is not responding then we just send the request again, maybe we get a response. Assume the following service that deposits 42 Swiss Francs (CHF) to account "1".

```
$ curl http://localhost:8080/accounts/1/deposits/f55cf209-a8bd-4b3e-b909-0ee8fc9dfd98
{"amount":42,"currency":"CHF"}
```

If this POST request fails for some reason (e.g. a timeout) and it is sent again then the money might be deposited twice. That is because in case of a timeout or other failures the client does not know if the recipient actually deposited the money or not. Maybe just the response was lost:

{% plantuml %}
participant "Service A" as A
participant "Network" as N
participant "Service B" as B
A -> N: send 42 CHF
activate N
N -> B: send 42 CHF
activate B
N x<-- B: OK, thanks
deactivate B
A -> A: Timeout

deactivate A
{% endplantuml %}

Of course it might also be the case that the transaction was not successfully processed. If the client does not retry the call then the money will never be added. So what can be done?

## How to design idempotent services

As mentioned before: For most services idempotence is important. It allows to retry calls that have failed without fearing any unwanted effects. It's amazing how many service I've seen in my daily work that are not idempotent. I had to make ugly workarounds in order to get things work due to services that were not idempotent. For example I implemented a service that hat to send (physical) letters to customers using a printing service. The printing service was not idempotent. So it was possible that a customer would receive the same letter twice or even more times. In order to avoid this problem I tried to work around it by checking whether there was a print job for the given customer already in the printing queue. This is not perfect because one millisecond after checking for a print job there might be created one by another process. So it would still be possible that a customer receives to letters.

Fortunately I see that awareness is increasing. Architectural patterns like [SAGA](https://microservices.io/patterns/data/saga.html)s are only possible if the services involved are idempotent. The increasing popularity of microservices are making it necessary to think more about properly designing services with idempotence in mind.

### A note on HTTP and REST

The following examples are RESTfull services (at least sort of... please don't call the REST police). However the general principles can also be applied to any other remoting protocol like [SOAP](https://en.wikipedia.org/w/index.php?title=SOAP&oldid=917411406), [gRPC](https://grpc.io/) etc.
 
One thing to note is the fact that HTTP defines which verbs (GET, POST, PUT, ...) are supposed to be idempotent and which are not. This is defined in [rfc7231](https://tools.ietf.org/html/rfc7231). When implementing a webservice this could of course be ignored but it would violate the principles of HTTP/REST. A REST service is not simply idempotent by using any of the "idempotent" verbs. It is idempotent because it was implemented this way.

- Idempotent: GET, HEAD, PUT, DELETE, OPTIONS and TRACE
- Non-Idempotent: POST, PATCH (see [rfc5789](https://tools.ietf.org/html/rfc5789))

An interesting comment in respect to idempotence and HTTP can be found [on Stackoverflow](https://stackoverflow.com/questions/45016234/what-is-idempotency-in-http-methods#targetText=A%20request%20method%20is%20considered,safe%20request%20methods%20are%20idempotent).


### Some services are already idempotent

Firstly it's important to note that read-only services are already idempotent.

{% highlight bash %}
$ curl http://localhost:8080/accounts/1/deposits/16dfa005-d882-4133-baf2-dfddbe7d61f0
{
  "amount" : 42,
  "currency" : "CHF"
}
{% endhighlight %}

This reads the deposit with ID "123". It does not change the deposit. Whether this is called once or twice does not matter. It will return the same deposit. Of course: if someone else modifies the deposit after the first call then the second call will return the modified deposit. But that is nothing to be solved by making the service idempotent. The point is that *the same client* is not changing the result by calling the service multiple times. In order to know whether someone changed the data it would be necessary to use versioning and/or locking techniques. That's a completely different story.

Assuming that existing deposits can be modified with a PUT request (the amount is changed to 120 CHF):

{% highlight bash %}
$ curl -X PUT -H "Content-Type: application/json" http://localhost:8080/accounts/1/deposits/c0f3279a-6774-4b2e-a98e-9c7e1bedbe32 -d '
{               
  "amount" : 120,
  "currency" : "CHF"
}'
{% endhighlight %}

Here we are changing the deposit "123" to have an amount of 120. This can be called multiple times without having a different outcome. So this conforms to HTTP which mandates PUT requests to be idempotent.


### Adding an id to the body

The problem with idemotence is mainly with services that create something. In the example a new deposits should be created for a given account. With HTTP this is usually done with POST:

{% highlight bash %}
$ curl-X POST http://localhost:8080/accounts/1/deposits -H "Content-Type: application/json" -d '
{ 
  "amount": 42, 
  "currency": "CHF" 
}'
aa564b53-6992-4577-8eae-c63bcafcb0a4
{% endhighlight %}

The creates a new deposit and returns the generated ID (aa564b53-6992-4577-8eae-c63bcafcb0a4). How can it be avoided that multiple invocations of this request won't create multiple resources? Here is a possible solution:

{% highlight bash %}
$ curl-X POST http://localhost:8080/accounts/1/deposits -H "Content-Type: application/json" -d '
{ 
  "id": 7
  "amount": 42, 
  "currency": "CHF" 
}'
{% endhighlight %}

So instead of letting the service generate an ID for the deposit the caller specifies the deposit ID (7) to be used in the body of the HTTP message. Now the service is able to distinguish whether the caller tries to repeatedly create the same deposit or if he tries to create a new deposit. 

One thing to consider is also what should happen if a client creates a POST with the same ID but a different deposit (different amount or also currency). The simplest approach is to just have a generic response saying something like "a deposit with this id has already been created" and ignore the request.

An issue with this solution is that the client decides which is the deposit ID to be used. Usually the service would like to choose the ID.


### Using PUT instead of POST for resource creation

By using PUT instead of POST it is possible to specify the deposit's ID in the URL:

{% highlight bash %}
$ curl -X PUT -H "Content-Type: application/json" http://localhost:8080/accounts/1/deposits/123 -d '
{               
  "amount" : 42,
  "currency" : "CHF"
}'
{% endhighlight %}

Similar to the POST example above the service is now able to decide whether he as already processed the transaction or not. Like with the POST example it is again the client who chooses the deposit ID (here 123).

If the deposit with id "123" does not exist yet, then it is created. Subsequent calls would update the existing deposit. This might work in some cases. However it would be more "RESTfull" to use POST for resource creation and PUT for updates.

### Providing an ID generator service

As mentioned before  a service would usually like to control the IDs used for its business objects. Also, if the client chooses the deposit ID then he could accidentally pick an ID that has been used by another client before. The service would then ignore this request and the client would never know. 

In order to avoid this, clients could use UUIDs ([preferably type 4, which are random](https://en.wikipedia.org/w/index.php?title=Universally_unique_identifier&oldid=915343615)). These UUIDs should be unique and there shouldn't be any collisions. The correct format of the ID should be validated in the service.

If the service must control the ID to be used then an ID generator service can be implemented:

{% highlight bash %}
$ curl-X POST http://localhost:8080/id-generator
{
  "id": "145b63ec-1440-46b5-b29f-6ae3c948dce4"
}
{% endhighlight %}

This id generator just generates a new ID each time it is called. This ID is then used when creating a deposit using POST. The service must check whether the deposit ID specified has been issued before. If this is true then the deposit will be created (if it does not exist yet).

The disadvantage of this solution is of course that the client has to perform two calls and an additional service (the id generator) has to be implemented.

### Providing a request id as request metadata

Currently I think that probably the best solution is to use a dedicated request id. In case of HTTP this could be implemented as a HTTP header (according to [this](https://www.keycdn.com/support/custom-http-headers) I'm not using an `x-` prefix). The request id is passed as a header using `-H "request-id: 123"` in curl.

{% highlight bash %}
curl -X POST http://localhost:8080/accounts/1/deposits -H "request-id: 123" -H "Content-Type: application/json"  -d '
{
  "amount": 42, 
  "currency": "CHF" 
}'
ca6cf316-6061-47a2-8948-6864d5b84839
{% endhighlight %}

Note that the service creates a new deposit id and returns it (ca6cf316-6061-47a2-8948-6864d5b84839). The service has to remember all `request-id`s that he successfully processed. If he repeatedly receives the same `request-id` he indicates this with a special return code. This solution has the advantage that the service can control the generation of the deposit ID and no additional service is required in order to generate an ID. Of course it is again up to the client to use a proper `request-id` that will not clash with IDs used by others. A good choice would be a UUID Type 4 and service side validation of the `request-id`.

This also allows to properly use POST instead of PUT for creation of resources.

## Service implementation considerations

When implementing and idempotent service special care has to be taken when persisting a `request-id`. One has to make sure the ID is "unique" (e.g. using UNIQUE constraints in a database). Violations of this constraint must be handled according to the approach taken.

Also it must be assured that the business entity (the account deposit in the example) is saved in the same local transaction as the `request-id`:

{% highlight java %}
@Path("/accounts/{id}")
public class AccountResource {
    @POST
    @Produces({MediaType.TEXT_PLAIN})
    @Path("/deposits")
    @Transactional
    public Response addDeposit(Deposit deposit, @HeaderParam("request-id") String requestId) {

        try {

            String newDepositId = UUID.randomUUID().toString();

            transactionTemplate.execute((status) -> {

                saveRequestId(requestId);
                saveNewDeposit(newDepositId, deposit);
                return null;
                
            });

            return Response.ok(newDepositId).build();
        } catch (Exception e) {
            if (requestIdExists(requestId)) {
                return Response.status(Response.Status.CONFLICT).build();
            }
            throw e;
        }
    }
}
{% endhighlight %}

In the example application ([see here for full source](https://github.com/olijaun/playground/tree/master/idemotence-example)) there are two inserts into the database performed: First the `requestId` is saved. Second the business entity itself (the deposit) is saved (the example uses a relational database and SQL for this). 

Both inserts are preformed inside the same local transaction (here using Spring programmatic transactions). This is very important: If one would insert the `requestId` in a separate transaction and the second insert for the deposit entry would fail afterwards then the request would appear to be processed on repeated calls.

This also shows that it is not possible to implement `saveIdempotenceId()` using a remote web service or using a different database. I mentioned that local transaction must be used. This is only possible "inside" the same database. It would be possible to use two different database and distributed transactions ([XA Transactions](https://de.wikipedia.org/wiki/X/Open_XA)). The problem with distributed transactions is that they are... well... distributed. So they are actually suffering kind of the same problem we are trying to solve. If you don't believe me then look-up "xa heuristic exceptions" ([look here for example](https://www.atomikos.com/Documentation/HeuristicExceptions)).





