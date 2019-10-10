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

So which test would you prefer as a lazy programmer? Of course I'm exaggerating a bit. As you can see Bean Validation seems to offer something called a ConstraintViolation. The EmailAddress doesn't have this "out-of-the-box" but it could certainly be implemented although I wouldn't do it. I'll explain later why.

Now, after bad-mouthing bean validation let's talk about it virtues.

# The virtues of Bean Validation

Assume the following RESTfull service:

{% highlight java %}
POST /registration
{
    "name": "Oliver Jaun",
    "emailAddress": "abc@example.com"
}
{% endhighlight %}