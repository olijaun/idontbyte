---
layout: test_post
title:  "Bean Validation is bad"
categories: [java, design]
comments: true
---

I've never really looked into [Java Bean Validation](https://jcp.org/en/jsr/detail?id=380) until some architects at my company suggested it. After going through a tutorial I was not convinced... So I googled for "Bean Validation is bad" in order to see if I'm the only one. And I was not...

One of the first blog posts that appeared was [this one here](https://in.relation.to/2014/06/19/blah-vs-bean-validation-you-missed-the-point-like-mars-climate-orbiter/) written by Emmanual Bernard which was the Spec Lead of Bean Validation. In his post he's is defending Bean Validation which is not surprising of course.

The guy who confirmed my doubts was a guy named Oliver Gierke (who seems to have a nice blog [here](http://olivergierke.de)). However this was just a comment in the comment section of the blog and I think this topic deserves a full blog post. I will also show hot to "properly" use Bean Validation. But let's start in the beginning.

# What's the problem?

I assume that you have some basic knowledge of bean validation (if not then [this article here](https://www.baeldung.com/javax-validation) might be a good introduction). Let's look at the example Oliver is using in his response to Emmanuel's post:

{% highlight java %}
class User {
    @NotNull @Email
    String emailAddress;
}
{% endhighlight %}

This assures that the email address cannot be null and that it is an email address. Note that in order for these rules to be applied you need to trigger somehow a validator that will do the actual validation (a common implementation would be [hibernate-validator](https://hibernate.org/validator/))

{% highlight java %}
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
Validator validator = factory.getValidator();
{% endhighlight %}

In case you're using JEE or some Spring extensions the frameworks might take care of the validation. So instead of triggering the validation yourself in an stateless session bean you could just write the following and the framework (e.g. JEE in this example) will take care of `User` validation. 

{% highlight java %}
@Stateless
public class MyBean {
    public void create(@Valid User user) {
        // ...
    }
}
{% endhighlight %}

The `@Valid` annotation makes sure the validation recurses to all fields that are annotated with @Valid. If you forget `@Valid` then nothing will be validated. 
 
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

So by using Bean Validation you are basically ignoring essential object oriented principals. Also you're tying your domain objects to a framework which is not really a good idea according to [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) the framwork is a "detail". So why would anyone invent something like Bean Validation? The specification states:

> To avoid duplication of these validations in each layer, developers often bundle validation logic directly into the domain model, cluttering domain classes with validation code that is, in fact, metadata about the class itself.

I do not agree with this. Validation logic is essential business logic! It is not meta data. You're domain logic expects something to be an "email address", a "contract number" or a "name" etc. also, Emmanual Bernard thinks that developers are lazy:

> If you do that in your projects kudos. Most developers including me are more lazy :)

I think it is the opposite: If they were lazy they would probably define an EmailAddress class. It is simpler to just create an `EmailAddress` object once and simply pass it around instead of adding validation annotations everywhere.

If you still think that it is a lot of work to create such objects then think about behaviour:

{% highlight java %}
EmailAddress emailAddress = new EmailAddress("james.bond@mi6.co.uk");

emailAddress.punyEncoded(); // returns a punny encoded string
emailAddress.isGmailAddress(); // returns true if email ends with gmail.com or google.com etc.
emailAddress.getDomainPart(); // returns "mi6.co.uk"
emailAddress.getLocalPart(); // returns "james.bond"
{% endhighlight %}

This is what objects are all about... We want to avoid an anemic domain model:

> The catch comes when you look at the behavior, and you realize that there is hardly any behavior on these objects, making them little more than bags of getters and setters [Martin Fowler (25 November 2003)](https://www.martinfowler.com/bliki/AnemicDomainModel.html)

It's amazing how Java developers try to avoid objects. I think the problem comes from the history of Java and frameworks like JEE and JPA that forced people to implement classes with empty constructors. In JPA you are forced to create DTOs which are not Objects but data structures. The "O" in "ORM" stands for Object but what you are doing is mapping data from the database to a data structure. There is [very funny blog post from Robert C. Martin (Uncle Bob)](https://blog.cleancoder.com/uncle-bob/2019/06/16/ObjectsAndDataStructures.html) where he writes about it. Don't miss it. 

It's also very simple to test the `User` object containing the EmailAddress object.

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

# How to "properly" use bean validation

You can find the "final solution" [in my github repository](https://github.com/olijaun/playground/tree/master/beanvalidation-example). Assume the following request to a registration service:

{% highlight bash %}
curl -X POST http://localhost:8080/registrations \ 
  -H "Content-Type: application/json" \
  -d '
{ 
  "emailAddress": "abc@def.ch", 
  "name": "Oliver Jaun", 
  "password": "secret" 
}'
{% endhighlight %}

There can be several validation error here. Now let's look at the webservice implementation (using jax-rs):

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

The `register` method receives a RegistrationDto. A Dto is not really an object it is a data structure. It is mutable and has getter and setter methods for modification. First the email address from the registrationDto is converted into an `EmailAddress` object. After this a `RegistrationCommand` is created that is passed to the `RegistrationService` which belongs to our domain model. The `RegistrationCommand` verifies that name, email address and password are not null. Note that the `RegistrationCommand` is created in the REST service implementation but it belongs and is defined in the domain model.

In the domain model we then have a RegistrationCommand that is valid and we can work with, without ever having to validate again. Note that the `RegistrationCommand` and the `EmailAddress` are immutable. So it is not possible that they suddenly are not valid anymore.

The problem with this is, that if the user passes an invalid email address then the `EmailAddress` will throw an `IllegalArgumentException` or a `NullPointerException`. Those are unchecked exceptions and jax-rs will create a HTTP response with the status code 500. However a RESTful service should respond with a 4xx status in such cases. So you could catch these errors and return the correct status code:

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
 
 Quite a lot of code just for an email address. Apart form that it would also be necessary to do something similar with the `RegistrationCommand`. This would be even more difficult because the `RegistrationCommand` contains three attributes. The name, the email address or the password... any of these could be wrong. How do we handle this properly?
 
 Assuming multiple things are wrong a UI would probably want to display all things at once. This code would get event more complicated because it would be necessary to collect multiple validation errors and return them summarized. That's where bean validation comes into play. First we add some annotations to the `RegistrationDto`:
 
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

It is fine to have Bean Validation annotations on DTOs because - as I said before - they are data structures not objects. Also because the DTOs do not live in the domain model so it is OK to tie them to a framework. In the jax-rs service implementation we just add `@NotNull` and `@Valid` annotation to the `register` method which is fine as well because you are using the jax-rs framework anyway.

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

Actually the bean validation will also throw unchecked exceptions. In order to not produce a status code 500 a ExceptionMapper is needed. An ExceptionMapper in jax-rs allows to map Exceptions to return codes. So here is a custom mapper that maps `ConstraintViolationException`s to an `ErrorDto`:

{% highlight java %}
@Provider
public class ConstraintViolationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {

    public Response toResponse(ConstraintViolationException e) {

        List<ValidationErrorDto> collect = e.getConstraintViolations().stream()
                .map(v -> new ValidationErrorDto(toLocation(v), v.getMessage())).collect(Collectors.toList());
        ErrorDto errorDto = new ErrorDto();
        errorDto.getErrors().addAll(collect);


        return Response.status(Response.Status.BAD_REQUEST).entity(errorDto).build();
    }

    private String toLocation(ConstraintViolation v) {
        String[] pathElements = v.getPropertyPath().toString().split("\\.");
        return Stream.of(pathElements).skip(2).collect(Collectors.joining("."));
    }
}
{% endhighlight %}

Note that I'm doing some fancy stuff with the path because the path contains the name of the java method. I think this name should not end up in an error message because it is implementation specific. Now let's see how this looks like when the service is invoked with invalid data:

{% highlight bash %}
curl -X POST http://localhost:8080/registrations \
  -H "Content-Type: application/json" \
  -d '
  {
    "name":"", 
    "emailAddress": "oliver@@jaun.org", 
    "password": "password123" 
  }'
  
{
  "errors" : [ {
    "location" : "name",
    "message" : "must not be blank"
  }, {
    "location" : "emailAddress",
    "message" : "must be a well-formed email address"
  } ]
}
{% endhighlight %}

This is nice: you can use the advantages of the Bean Validation framework but your domain model (EmailAddress etc.) stays clean. In your domain model you can rely on the objects to be properly validated. Look at the RegistrationService in the domain model:

{% highlight java %}
public class RegistrationService {

    public RegistrationId register(RegistrationCommand registrationCommand) {

        // perform registration

        return registrationId;
    }
}
{% endhighlight %}

There is no bean validation at all and we can implement our domain logic without having to worry about anything. The RegistrationCommand will be valid because the constructor of the RegistrationCommand will make sure it is. Bean Validation is just a detail that is used outside of the domain logic in order to provide nice user error message to the user.

But what happens if we forget to add the `@Valid` annotation? In this case the RESTful service still returns status code 500 because the instantiation of the domain object will throw an unchecked exception (e.g. `IllegalArgumentException`). This is fine. Firstly: it should not happen if you write tests. Secondly if it happens then it's a bug. Whenever the service returns status code 500 this should be logged. The error must be analyzed and then fixed as you do with any other bug.

Note that the example could still be improved. IMHO there should be no plain error messages be returned by a REST service. I'm working for a swiss company and we have to support german, french and italian. What does the service return? German? French? English? All of them? Also an application calling this service might not just display a message but react to accordingly in case of an error. So I think it would be better to return some kind of error code instead of a plain error message.

# Some more thoughts

It could be argued that validation logic is now implemented twice. Once in the implementation for the `@Email` annotation and once in the domain model by the `EmailAddress` class. Yes this is true but often the validation logic can be shared. Let's look at an extended version of the `EmailAddress` class:

{% highlight java %}
public class EmailAddress {
    
    public static final String EMAIL_ADDRESS_PATTERN = "[^@]+@[^@]+";
    private String emailAddress;

    public EmailAddress(String emailAddress) {
        this.emailAddress = Objects.requireNonNull(emailAddress);

        if(!emailAddress.matches(EMAIL_ADDRESS_PATTERN)) {
            throw new IllegalArgumentException("invalid email address: " + emailAddress);
        }
    }

    public String asString() {
        return emailAddress;
    }
}
{% endhighlight %}

This is not a proper email address pattern, I agree. Email address validation is quite difficult. For the sake of demonstration I still use this simplified pattern.

As stated above one of my objections with bean validation is that your domain model should not depend on it. However it is fine if your application- or infrastructure layer depends on the domain logic. In order to not duplicate the validation we can reuse the EmailAddress validation in the RESTful service implementation. One possibility is to just use the `EMAIL_ADDRESS_PATTERN` defined by the `EmailAddress` class in the `RegistrationDto`:

{% highlight java %}
public class RegistrationDto implements Serializable {

    @NotNull
    @Pattern(regexp = EmailAddress.EMAIL_ADDRESS_PATTERN)
    private String emailAddress;
    @NotNull
    @NotBlank
    private String name;
    @NotNull
    @NotBlank
    private String password;
}
{% endhighlight %}

So instead of the `@Email` annotation the `@Pattern` annotation is used which accepts a regular expression. This is a pragmatic approach in many situations. If the validation cannot be expressed by a regular expression, then another solution would be to create a new bean bean validation annotation:

{% highlight java %}
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(validatedBy = { EmailAddressValidator.class })
public @interface ValidEmailAddress {
    String message() default "invalid email address {value}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
{% endhighlight %}

Then the `EmailAddressValidator` has to be created, which basically delegates the validation to our `EmailAddress` class.

{% highlight java %}
public class EmailAddressValidator implements ConstraintValidator<ValidEmailAddress, String> {

    @Override
    public void initialize(ValidEmailAddress ageValue) {
    }

    @Override
    public boolean isValid(String emailAddressAsString, ConstraintValidatorContext constraintValidatorContext) {
        // null values are valid
        if (emailAddressAsString == null) {
            return true;
        }

        try {
            new EmailAddress(emailAddressAsString);
            return true;
        } catch (IllegalArgumentException e) {
            return false;
        }
    }
}
{% endhighlight %}

# Summary

Bean Validation is not as bad as long as it is NOT used in the domain model of an application.

The proposed approach of combining bean validation with proper domain objects that validate themselves has several advantages:

Domain Objects remain "clean" (they do not depend on a framework) and can simply throw `NullPointerException`s or `IllegalArgumentException`s in case that validations fails in the constructor.

Bean Validation is used where it "shines". It is applied only on the outer layer of an application and allows for validation with detailed messages. It also allows for multiple validation errors to be returned at once.

The disadvantage of the approach is that there is a certain redundancy in validation which can however be minimized by sharing validation code.