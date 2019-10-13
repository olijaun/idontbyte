---
layout: test_post
title:  "Why Bean Validation is evil and how it could be useful anyway"
categories: [java, design]
comments: true
---

I've never really looked at [Java Bean Validation](https://jcp.org/en/jsr/detail?id=380)) until some architects at my company suggested it. After going through a tutorial I was not convinced... So I googled for "Bean Validation is bad" in order to see if I'm the only one. And I was not...

However one of the first blog posts that appeared was [this one here](https://in.relation.to/2014/06/19/blah-vs-bean-validation-you-missed-the-point-like-mars-climate-orbiter/) written by Emmanual Bernard which was the Spec Lead of Bean Validation. He's is actually defending Bean Validation in the article which of course is not a surprise.

The guy who confirmed my doubts was a guy named Oliver Gierke (who seems to have a nice blog [here](http://olivergierke.de)). His comments on the mentioned blog post were very good and I can agree almost 100% with what he is saying. However this was just a small comment and I think this topic deserves a full blog post. I also found a a way of using Bean Validation in an acceptable way that I would like to share. But let's start in the beginning.

# What's the problem?

Let's look at the example Oliver is using in his response to Emmanuel's post:

{% highlight java %}
class User {
	@NotNull @Email
	String emailAddress;
}
{% endhighlight %}

This assures that the email address cannot be null and that it conforms to some other validation rules defined by `@Email`. So if you have e.g. an ejb bean method that accepts a user you have to do the following:

{% highlight java %}
@Stateless
public class MyBean {
    public void create(@Valid User user) {
        // ...
    }
}
{% endhighlight %}

If you forget @Valid then nothing will be validated. If this would be a method of an unmanaged class (just some class you're instanciating) then no validation would be performed(regardless of whether you are using `@Valid` or not)!

Let's look at an alternative approach Oliver Gierke is proposing:

{% highlight java %}
class EmailAddress {
	
	public EmailAddress(String value) {
	  // explicit null check
	  // regex validation
	}
}

class User {
	
	public User(EmailAddress email) {
	  // reject null email
	}
}
{% endhighlight %}

A new class `EmailAddress` is introduced. The constructor explicitly validates the value. In other words: the constructor is doing what it was made for:

>  They \[constructors\] have the task of initializing the object's data members and of establishing the invariant of the class, failing if the invariant is invalid. [Wikipedia](https://en.wikipedia.org/w/index.php?title=Constructor_(object-oriented_programming)&oldid=919933748)

So by using Bean Validation you are basically ignoring essential object oriented principals. So why would anyone invent something like Bean Validation? Emmanual Bernard thinks that developers are lazy:

> If you do that in your projects kudos. Most developers including me are more lazy :)

I think it is the opposite: If they were lazy they would define an EmailAddress class. Let's look at the following code where we use the first version of the User object that contains an email as string:

{% highlight java %}
@Stateless
public class MyBean {
    public void create(@Valid User user) {
        // ...
        sendEmail(user.getEmailAddress())
    }
}

@Stateless
class EmailSender {
    public void sendEmail(@Email String emailAddress) {
        //...
    }
}
{% endhighlight %}

The `create` method calls `sendEmail` in another ejb. Again the email has to be validated because it is just a string. Now imagine how it looks when EmailAddress is a proper object:

{% highlight java %}
@Stateless
public class MyBean {
    public void create(User user) {
        // ...
        sendEmail(user.getEmailAddress())
    }
}

@Stateless
class EmailSender {
    public void sendEmail(EmailAddress email) {
        //...
    }
}
{% endhighlight %}

Although the difference is not very big, the second version looks simpler to me. Of course, you have to create an extra object. But then you could also implement some behaviour...

{% highlight java %}
EmailAddress emailAddress = new EmailAddress("james.bond@mi6.co.uk");

emailAddress.punyEncoded(); // returns a punny encoded string
emailAddress.isGmailAddress(); // returns true if email ends with gmail.com or google.com etc.
emailAddress.getDomainPart(); // returns "mi6.co.uk"
emailAddress.getLocalPart(); // returns "james.bond"
{% endhighlight %}

This is what objects are all about. 

It's amazing how Java developers try to avoid objects. I think the problem comes from the history of Java and frameworks like JEE and JPA that forced people to implement classes with empty constructors. In JPA you are forced to create DTOs which are not Objects but data structures. The "O" in "ORM" stands for Object but what you are doing is mapping data from the database to a data structure. There is [very funny blog post from Robert C. Martin (Uncle Bob)](https://blog.cleancoder.com/uncle-bob/2019/06/16/ObjectsAndDataStructures.html) where he writes about it. Don't miss it. 

It's also very simple to test the User Object containing the EmailAddress object.

{% highlight java %}
class Test {
  @Test
  void invalid() {
    
    Execution e = () -> new User(new EMailAddress("invalid@@@@@@email.com"));
    
    assertThrows(IllegalArgumentException.class, e);
  }
}
{% endhighlight %}

Pretty simple, right? Now with Bean Validation:

{% highlight java %}
class Test {
  @Test
  void invalid() {
    User user = new User("invalid@@@@@@email.com");
    
    ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
    Validator validator = factory.getValidator();
    Set<ConstraintViolation<AdditionBean>> errors = validator.validate(bean);
    assertTrue(errors.size() > 0);
  }
}
{% endhighlight %}

So which test would you prefer as a lazy programmer? Of course I'm exaggerating a bit. As you can see Bean Validation seems to offer something called a ConstraintViolation. The EmailAddress doesn't have this "out-of-the-box" but it could certainly be implemented although I would not do it. I'll explain later why.

Now, after bad-mouthing bean validation let's talk about its virtues.

# The virtues of Bean Validation

Assume the following request to a registration service:

{% highlight java %}
POST /registrations
{
    "name": null,
    "emailAddress": "abc@@@@@@@@example.com"
    "password": "mypassword"
}
{% endhighlight %}

There are three errors in the input. First the name is `null`. Second the email address is not valid and third the password is not valid because it should contain a number. Now let's look at the webservice implementation (using jax-rs):

{% highlight java %}
@Path("/registrations")
public class RegistrationResource {

    @Inject
    private RegistrationService registrationService;

    @POST
    @Produces({MediaType.TEXT_PLAIN})
    @Path("/")
    public Response register(RegistrationDto registrationDto) {

        EmailAddress emailAddress = new EmailAddress(registrationDto.getEmailAddress());

        RegistrationCommand registrationCommand
                = new RegistrationCommand(registrationDto.getName(), emailAddress, registrationDto.getPassword());

        RegistrationId registrationId = registrationService.register(registrationCommand);

        return Response.ok(registrationId.asString()).build();
    }
}
{% endhighlight %}

The `register` method receives a RegistrationDto. A Dto is not really an object it is a data structure. It is mutable and has getter and setter methods for modification. First the email address from the registrationDto is converted into an `EmailAddress` object. After this a `RegistrationCommand` is created that is passed to the `RegistrationService` which belongs to our domain model. The `RegistrationCommand` verifies that name, email address and password are not null. 

In the domain model we then have a RegistrationCommand that is valid and we can work with without ever having to validate again. Note that the `RegistrationCommand` and the `EmailAddress` are immutable. So it is not possible that they suddenly are not valid anymore.

The problem with this is, that if the user passes an invalid email address then the `EmailAddress` will throw an `IllegalArgumentException` or a `NullPointerException`. Those are unchecked exceptions and jax-rs will create a HTTP response with the status 500. However a RESTful service should reppond with a 4xx status in such cases. So you could catch these errors and return the correct status code.

 {% highlight java %}
 @Path("/registrations")
 public class RegistrationResource {
 
     @Inject
     private RegistrationService registrationService;
 
     @POST
     @Produces({MediaType.TEXT_PLAIN})
     @Path("/")
     public Response register(RegistrationDto registrationDto) {
 
         EmailAddress emailAddress;
         try {
             emailAddress = new EmailAddress(registrationDto.getEmailAddress());
         } catch(IllegalArgumentException e) {
             return Response.status(Response.Status.BAD_REQUEST).entity("invalid email address format").build();
         } catch (NullPointerException e) {
             return Response.status(Response.Status.BAD_REQUEST).entity("email address is mandatory").build();
         }
 
         RegistrationCommand registrationCommand
                 = new RegistrationCommand(registrationDto.getName(), emailAddress, registrationDto.getPassword());
 
         RegistrationId registrationId = registrationService.register(registrationCommand);
 
         return Response.ok(registrationId.asString()).build();
     }
 }
 {% endhighlight %}
 
 Quite a lot of code just for an email address. Apart form that it would also be necessary to do something similar with the `RegistrationCommand`. This would be even more difficult because the `RegistrationCommand` manages three attributes. The name, the email address or the password... any of these could be wrong. How do we handle this properly?
 
 Assuming multiple things are wrong a UI would probably want to display all things at once. This code would get event more complicated because it would be necessary to collect multiple validation errors and return them summarized in the end. That's where bean validation comes into play. First look at the RegistrationDto:
 
{% highlight java %}
public class RegistrationDto implements Serializable {

    @Email
    private String emailAddress;
    @NotNull @NotBlank
    private String name;
    @NotNull @NotBlank
    private String password;

    // setters and getters
}
{% endhighlight %}

It is fine to have Bean Validation on DTOs because - as I said before - they are data structures not objects. In the jax-rs service implementation we just add `@NotNull` and `@Valid` annotation to the `register` method which is fine as well because you are using the jax-rs framework anyway.

{% highlight java %}
@Path("/registrations")
public class RegistrationResource {

    @Inject
    private RegistrationService registrationService;

    @POST
    @Produces({MediaType.TEXT_PLAIN})
    @Path("/")
    public Response register(@NotNull @Valid RegistrationDto registrationDto) {

        EmailAddress emailAddress = new EmailAddress(registrationDto.getEmailAddress());

        RegistrationCommand registrationCommand
                = new RegistrationCommand(registrationDto.getName(), emailAddress, registrationDto.getPassword());

        RegistrationId registrationId = registrationService.register(registrationCommand);

        return Response.ok(registrationId.asString()).build();
    }
}
{% endhighlight %}

This is nice: you can use the advantages of the Bean Validation framework but your domain logic classes (EmailAddress etc.) remain clean. In your domain model you can rely on the objects to be properly validated. Look at the RegistrationService in the domain model:

{% highlight java %}
public class RegistrationService {

    public RegistrationId register(RegistrationCommand registrationCommand) {

        RegistrationId registrationId = new RegistrationId(UUID.randomUUID());

        // perform registration

        return registrationId;
    }
}
{% endhighlight %}

No bean validation at all and we can implement our domain logic without having to worry about anything. The RegistrationCommand will be valid not because we are using bean validation. It will be valid because the constructor of the RegistrationCommand will make sure it is.

But what happens if we forget to add the `@Valid` annotation? In this case the RESTful service will still return status code 500 because the instantiation of the domain object will throw an unchecked exception (e.g. `IllegalArgumentException`). This is fine. Firstly: it should not happen if you write tests. Secondly if it happens then it's a bug. Whenever the service returns status code 500 this should be logged. The error must be analyzed and then fixed as you do with any other bug.

It event get more interesting when using the `ConstraintValidations` Bean Validation is offering. It is possible to write a jax-rs `ExceptionMapper` that returns converts and returns these constraint validations in a structured manner. Probably I'll write aboute this in another blog post.

# Summary

Bean Validation is not as bad as long as it is NOT used in the domain model of an application.

The proposed approach of combining bean validation with proper domain objects that validate themselves has many advantages:

Domain Objects remain "clean" and can simply throw `NullPointerException`s or `IllegalArgumentException`s in case that validations fails in the constructor.

Bean Validation is used where it "shines". It is applied only on the outer layer of an application and allows for validation with detailed messages. It also allows for multiple validation errors to be returned at once.

The disadvantage of the approach is that there is a certain redundancy in validation which can however be minimized by reusing validation code in some cases.