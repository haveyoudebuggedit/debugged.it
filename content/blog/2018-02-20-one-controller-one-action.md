---
slug: one-controller-one-action
guid: https://pasztor.at/blog/one-controller-one-action/
authors:
- janos
categories:
- Technology
date: "2018-02-20T00:00:00Z"
publishDate: "2018-02-20T00:00:00Z"
summary: How many actions do you put in a controller? 5-6? 20? What would you say if I told you my limit is just one method?
images:
- posts/one-controller-one-action.png
preview: posts/one-controller-one-action.jpg
tags:
- Software Development
title: One Controller, One Action
---

It is safe to say that most web applications have way too many action methods in their controllers. It starts out with
four-five, but it quickly grows out of control and becomes
[a walking-talking Single Responsibility Principle violation](/blog/clean-code-responsibilities). I have been talking
with a friend about this problem and they suggested that the approach of putting only one action in one controller class
may be a solution to that. As preposterous as that sounds, let's just follow this trail for a minute.

## One controller...

A very popular approach towards structuring controllers is the separation along the lines of CRUD
(Create-Read-Update-Delete). If we were to write a very simple API to work with `BlogPost` entities following this
methodology, we would get something like this: 

```java
class BlogPostController {
    @Route(method="POST", endpoint="/blogposts")
    public BlogPostCreateResponse create(String title /*...*/) {
        //...
    }

    @Route(method="GET", endpoint="/blogposts")
    public BlogPostListResponse list() {
        //...
    }

    @Route(method="GET", endpoint="/blogposts/:id")
    public BlogPostGetResponse get(String id) {
        //...
    }
    
    @Route(method="PATCH", endpoint="/blogposts/:id")
    public BlogPostUpdateResponse update(String id, String title /*...*/) {
        //...
    }

    @Route(method="DELETE", endpoint="/blogposts/:id")
    public BlogPostDeleteResponse delete(String id) {
        //...
    }
}
```

On the surface this looks good since all functionality pertaining to the `BlogPost` entity is grouped together.
However, we have left one part out: the constructor. If we use dependency injection (which we really should), our 
constructor must declare all dependencies like this:

```java
class BlogPostController {
    private UserAuthorizer userAuthorizer;
    private BlogPostBusinessLogic blogPostBusinessLogic;
    
    public BlogPostController(
        UserAuthorizerInterface userAuthorizer,
        BlogPostBusinessLogicInterface blogPostBusinessLogic
    ) {
        this.userAuthorizer        = userAuthorizer;
        this.blogPostBusinessLogic = blogPostBusinessLogic;
    }
    
    /* ... */
}
``` 

Now, let's *write a test*. You do test your applications, right? First, our test for the `get` method:

```java
class BlogPostControllerTest {
    @Test
    public void testGetNonExistentShouldThrowException() {
        BlogPostController controller = new BlogPostController(
            //get does not need an authorizer
            null,
            new FakeBlogPostBusinessLogic()
        );
        
        //Do the test
    }
}
```

Wait... did you see that? The first parameter of the constructor is `null`. You may be thinking, *so what*? But this 
is really important: the `null` indicates that your controller has a dependency that is not needed for the `get()`
method.

I would stipulate that if this is the case, you have a Single Responsibility Principle violation, since you can remove
that dependency without affecting the functionality of the `get()` method.

True, single responsibility is defined in a business sense, not in a coding sense, but chances are you are also
violating SRP in a business sense if you follow the CRUD setup.

{{% tip %}}
**Single Responsibility Principle:** A class should have only one reason to change.
{{% /tip %}}

## ...one action

When I started looking at my code with this realization, I had to admit: no CRUD-style or other controller held up to
closer inspection when looking for SRP violations.

So, I propose a radical solution: *one controller, one action*. After refactoring, our code would look like this:

```java
class BlogPostGetController {
    private BlogPostBusinessLogicInterface blogPostBusinessLogic;
    
    public BlogPostGetController(
        BlogPostBusinessLogicInterface blogPostBusinessLogic
    ) {
        this.blogPostBusinessLogic = blogPostBusinessLogic;
    }
    
    @Route(method="GET", endpoint="/blogposts/:id")
    public BlogPostGetResponse get(String id) {
        //...
    }
}
```

Simple, nicely packed, and most of all: responsibilities don't get any more single-er than this. But wait, there's more!
Look at the `BlogPostBusinessLogicInterface`. Judging from the API that must also have a fair-few methods. There is a
little something called the Interface Segregation Principle. 

{{% tip %}} 
**Interface Segregation Principle:** No client (caller) should be forced to depend on methods it does not use.
{{% /tip %}}

If we want to adhere to this principle, we need to split that interface up into `BlogPostGetBusinessLogicInterface`
and a couple of others. The implementation could then look like this:

```java
class BlogPostBusinessLogicImpl
    implements
        BlogPostGetBusinessLogicInterface,
        BlogPostCreateBusinessLogicInterface,
        /* ... */ {
        
    /* ... */
}
```

However, this class probably suffers from the same problems as our controller: it's the embodiment of a Single
Responsibility Principle violation. The business logic to *get* a blog post and to *create* one are fundamentally
different.

In order to resolve this issue we can apply the same approach as with the controller: split up the (*probably several
thousand line long*) `BlogPostBusinessLogicImpl` into neatly packaged, single-method classes.

Then we continue on to the data storage layer, and find the same thing happening there as well. So we split the
interfaces as well as the implementation itself.

If we follow this logic through, you end up with an application that's cut into classes that have only one action. But
while we're at it, we can push things just a little bit further.

## Is this... functional?!

If you squint a little you may see a strange pattern emerge: Our constructors have the sole purpose of storing the
incoming dependency, in our case the `blogPostBusinessLogic` object, in an instance variable. The
`blogPostBusinessLogic` itself is also a class instance with a single *function* in it, which will be used by the action
during the execution. 

As we will see in this section, a class with only one constructor and one method is very similar to the combination of
two concepts used in *functional programming*: **higher order functions** and **currying**.

A *higher order function* is one that takes a different function as an argument. A simple example in
JavaScript would look like this:

```javascript
//foo gets bar (a function) as a parameter for execution
function foo (bar) {
    //The function stored in the variable bar is executed and the result returned
    return bar();
}
```

*Currying* occurs when we split up a function with, say, two parameters into a function with one parameter that
returns a second function with again, one parameter.

So this:

```javascript
function add (a, b) {
    return a + b;
}
//yields 42
add(32, 10);
```
 
Becomes this:

```javascript
function add (a) {
    return function(b) {
        return a + b;
    }
}
//yields 42
intermediate = add(32);
final = intermediate(10);
```

Currying allows for greater separation of concerns, since the first call can be in a completely separate part of the 
application than the second.

Joining *higher order functions* and *currying*, our previous Java code can be rewritten in functional-style Javascript
like this:

```javascript
/**
 * This is the constructor, which is receiving the dependencies.
 * 
 * @param {function} blogPostGetBusinessLogicInterface
 */
function BlogPostGetController(blogPostGetBusinessLogicInterface) {
    /**
     * This is the actual action method. 
     * 
     * @param {string} id
     */
    return function(id) {
        //Call the business logic passed in the outer function.
        //(analogous to the getById method)
        return blogPostGetBusinessLogicInterface(id)
    }
}
```

If you observe carefully, the functional-style Javascript and the OOP Java implementation has the same functionality,
which is to fetch the blog post from the business logic and return it.

So in essence, having only one action per controller *brought us closer to writing functional code*, as the
single-method class behaves almost exactly how a higher order function would behave. You can still keep writing OOP
code and use some beneficial aspects of functional programming as well.

But we can go even further, we can actually make our Java code *pure*. (A pure function has no mutable state in it.) To
achieve this, we declare all variables `final` so they can't be modified once set:

```java
class BlogPostGetController {
    private final BlogPostGetBusinessLogicInterface blogPostGetBusinessLogic;
    
    public BlogPostGetController(
        final BlogPostGetBusinessLogicInterface blogPostGetBusinessLogic
    ) {
        this.blogPostGetBusinessLogic = blogPostGetBusinessLogic;
    }
    
    @Route(method="GET", endpoint="/blogposts/:id")
    public BlogPostGetResponse get(final String id) {
        //All variables here should be final
        return new BlogPostGetResponse(
            blogPostGetBusinessLogic.getById(id) 
        );
    }
}
```

Cool! Now we are hip and are doing functional programming in Java! Well, more or less anyway. Functional-style
programming doesn't make your code magically better. You can still write methods that are thousands of lines long, but
it will be just a tiny bit harder.

## It's not OOP vs FP

Many discussions on the internet seem to be going along the lines of OOP being the mortal enemy of functional
programming, FP being either the *future of programming* or a *hipster fad*, depending on which side you listen to.

However, the truth is that [OOP and FP get along just fine](/blog/functional-object-oriented-programming).
Object orientation gives you structure, whereas functional programming gives you immutability and easier to test code.

The one controller one action paradigm, when combined with immutability, leads to a beneficial blend of
OOP and FP in my opinion.

{{% tip %}}
**Recommended resources:**

- [Why functional programming matters (video)](https://www.youtube.com/watch?v=oB8jN68KGcU)
- [The Entity-Boundary-Interactor pattern](http://ebi.readthedocs.io/en/latest/)
- [The Action Domain Responder pattern](https://github.com/pmjones/adr)
{{% /tip %}}
