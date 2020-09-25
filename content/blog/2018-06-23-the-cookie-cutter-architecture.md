---
slug: the-cookie-cutter-architecture
guid: https://pasztor.at/blog/the-cookie-cutter-architecture/
authors:
- janos
categories:
- Technology
date: "2018-06-23T00:00:00Z"
publishDate: "2018-06-23T00:00:00Z"
summary: When it comes to business applications, you need an architecture that scales well. This is my take, based on Uncle Bobs EBI.
images:
- posts/cookie-cutter-architecture.png
preview: posts/cookie-cutter-architecture.jpg
tags:
- Software Development
title: The Cookie Cutter Architecture
---

Even though most people recognize me as a DevOps person, I often work with business applications as well during the
course of my consulting projects or even while writing management software for my DevOps ventures. During my years of 
doing that I realized that my way of writing code wasn't terribly efficient.

At first I'd start out with a framework, such as Symfony, (not bashing on Symfony,) and I would write my code in a way
as it is exemplified in the Symfony documentation. However, the Symfony documentation contains simplified examples
on how to do things rather than the gold standard. For example, it would have database (Doctrine) queries directly 
in the controller.

Many try to rationalize this as being *simple* and that Doctrine is the *model* in MVC, but as I came to learn, having
a flat architecture just doesn't scale well. As the requirements grew the controller would become more and more bloated,
and common code parts would not be separated out. This was a problem, but I knew no solution to it.

A couple of years back I had come across Uncle Bobs talk [Architecture - The Lost Years](https://www.youtube.com/watch?v=o_TH-Y78tt4),
but it was way too academic, way too theoretical. Even though there is a sizable amount of documentation on his
proposed setup called [Entity-Boundary-Interactor](https://ebi.readthedocs.io/), I found it to be too simplistic.

Then, about two years ago, I switched from PHP to Java. Not because I hated PHP, far from it. I just wanted to have deep
[static typing](/blog/loose-strict-static) which PHP was (and still is) lacking. I first switched to Hacklang, which was
wonderful but lacked any sort of reasonable IDE support at the time. Finally I gave up and ported all my code to Java.

Looking at the Java world I didn't like it too much. As a newcomer to the language I disliked the, in my eyes,
archaic Servlet API and missed the by comparison modern immutable HTTP representations set forth in
[PSR-7](https://www.php-fig.org/psr/psr-7/).

Since I was under no time pressure I did what no sane person in a business environment would do and rolled my own. I 
ported PSR-7 to Java, and wrote a mapper to the servlet API. I build an abstraction around Jetty to serve as an embedded
web server, so I was free from the constraints of the usual Java architectures, free to build and experiment with any
system I liked.

Around summer last year an other concept started to creep into my view that heavily influenced how I built my systems:
Single Page Applications. Hate or love React and its buddies, I started thinking about my application more like an
API than something messy that has to deal with *state* in *sessions*, storing form data and what not.

I also heavily built on the concept of [dependency injection](/blog/clean-code-dependencies) using my home-made
[dependency injector](https://github.com/opsbears/owc-dic). But most importantly, I did a lot of thinking on how one
can build an application that is well maintainable down the line.

## The Architecture

As a result of my experiments I raise you one architecture that I came up with. The emphasis is *not* on writing the
least amount of code. It is also *not* on being the fastest to write. The main point is *predictability*. In other
words, I hate surprises. When something tiny changes in the requirements, it should not ripple through the entire
application, messing everything up. A third party API change or bug should be caught in one place and should not cause
a cascade failure.

To achieve that my application is split into three layers: the *API*, the *business logic*, and the *backend/storage*.
Each of these layers is responsible for one thing only and they pass *entities* between them. (Basically, semi-dumb
data transfer objects).

### The API

The **API** is responsible for dealing with the output medium. For example, if an API needs to return a blog post, but
wants to also fetch the author of said blog post, it needs to call the appropriate business logic classes to do that.
For example:

```java
class BlogPostGetApi {
  private final BlogPostGetBusinessLogic blogPostGetBusinessLogic;
  private final AuthorGetBusinessLogic authorGetBusinessLogic;
  
  @Inject
  public BlogPostGetApi(
    BlogPostGetBusinessLogic blogPostGetBusinessLogic,
    AuthorGetBusinessLogic authorGetBusinessLogic
  ) {
    this.blogPostGetBusinessLogic = blogPostGetBusinessLogic;
    this.authorGetBusinessLogic = authorGetBusinessLogic;
  }
  
  @Route(
    method = "GET"
    path = "/blog/:id"
  )
  public Response get(String id) throws BlogPostNotFoundException {
    BlogPost blogPost = blogPostGetBusinessLogic.getById(id);
    Author author = authorGetBusinessLogic.getById(blogPost.getAuthorId());
  
    return new Response (
      blogPost,
      author
    )
  }
  
  //Inner class that contains a structured response
  public class Response {
    //...
  }
}
```

As you can see the business logic will not have to deal with the complexities of which objects to fetch, that is
for the API layer to deal with. It is also important for the API to deal with any potential permission issues, such as:

```java
if (!blogPost.isPublished()) {
  throw new BlogPostNotFoundException();
}
```

It gets more complicated, but you get the idea. (We'll talk about permission checking a little further down.)

### The Business Logic

The business logic, in contrast is responsible for **how** you do something. So you could have classes such as
`UserCreateBusinessLogic`. This class is **only** responsible for the business process that leads to a user creation.
Any API that needs to create a user for whatever reason, can depend on the `UserCreateBusinessLogic` to make sure
user creation is done in one place and one place only.

Needless to say, business processes can get complicated, so they can, of course, call each other. For example, if you
had a business process that creates an *Organization* object, and a user for it, you could have a
`UserOrganizationRegisterBusinessLogic`, which would call both the `UserCreateBusinessLogic` and the
`OrganizationCreateBusinessLogic`. Maybe also the payment creation and a couple of others.

### The backend/storage

The final layer in our application is responsible for dealing with any of the pesky parts *outside* of our
applications, such as third party APIs, databases, and all other stuff that we would treat as *unreliable*.

... wait a minute ... did I just say a database is unreliable? Yepp, I just said that. The database is on a network,
and as much as developers would like it to be otherwise, networks are unreliable. They can go out, they can be slow,
they can have packet loss. So I treat a database the same as I treat a third party API.

So back to APIs... often times we have to deal with third party APIs that we don't really know all that well. Either
it is underdocumented, or it just has some quirks that we haven't encountered yet. These will inevitably cause problems
in our system that we have to deal with sooner or later, such as catching errors and, for example, letting the
business logic now that *I can't do this right now*.

For example, the data structure may be *weird* and not to our liking, in which case it's the backend layers job to
transform that into an object that we can then work.

### Entities

{{% tip %}}
**Note:** This article does not differentiate between entities and DTOs. For the purposes of this article an
entity is a structured set of data that you want to pass between parts of your application. If you want to split 
them based on purpose or use, go for it.
{{% /tip %}}

As I mentioned the different layers communicate using *entities*. These are not entities in the sense that you would
expect from an ORM system. They do not contain magic functions to load subobjects, like `blogPost.getAuthor()`. These
are dumb data transfer objects such as this:

```java
class BlogPost {
  private final String id;
  private final String authorId;
  private final String title;
  
  public BlogPost(
    String id,
    String authorId,
    String title
  ) {
    this.id = id;
    this.authorId = authorId;
    this.title = title;
  }
  
  public String getId() {
    return this.id;
  }
  
  //...
}
```

Want to fetch the author belonging to this blog post? Do it yourself. In my eyes it should be explicit in your business
logic. Readable code to document what's happening instead of relying on an ORMs internal behavior.

You may also notice that the entity above is *immutable*. If you want to modify the title, you have to do so in a copy.
For this purpose the entity may contain a helper function:

```java
public BlogPost withTitle(
  String title
) {
  return new BlogPost(
    this.id,
    this.authorId,
    title
  );
}
```

And that's it! There isn't much more that goes into entities, apart from *maybe* validation. After all, an entity with
invalid data should not even be created, failure should happen early.

## Dealing with consistency

Remember that organization and user registration example we had above? One registration process deals with creating
multiple objects, which in turn, use multiple storage classes to save the data to a database.

How do you ensure consistency? In other words, how do you ensure that either all of them or none of them are created?

That's where Java really starts to shine. There's a thing called the [Java Transaction API](https://en.wikipedia.org/wiki/Java_Transaction_API),
which allows creating *distributed* transactions, **even across multiple databases**.

I simply request a `Transaction` object in my API layer when doing an operation that requires it, and then pass it
through my application down to the storage layer. The storage layer can then use it to ensure consistency, even across
multiple object creations / updates.

## Dealing with permission checking

I have struggled for implementing more complex permission checks for a very long time. The API layer itself is not 
suitable for implementing extensive permission checks as those may need to be reused across multiple APIs.

Let's take a very simple example: every user who logs in gets an access token and then you request said access
token in the API for every request requiring permissions like this:

```java 
public Response update(
  @RequestHeader(name = "Authorization", prefix = "Bearer") 
  String accessToken,
  String blogPostId,
  String title,
  //...
) {
  //...
}
```

In this example the Single Page Application would send the access token in the `Authorization` header like this:

```
Authorization: Bearer your-access-token-here
```

However, your API needs to determine if the user logged in with said access token has permissions to update this blog
post. We can employ a little trick here: we add an additional security layer in front of the business logic, such
as this:

```java
public Response update(
  @RequestHeader(name = "Authorization", prefix = "Bearer") 
  String accessToken,
  String blogPostId,
  String title,
  //...
) throws AccessDeniedException, BlogPostNotFoundException {
  newBlogPost = blogPostUpdateSecurity.update(
    blogPostId,
    accessToken,
    title,
    //...
  );
  return new Response(
    newBlogPost
  );
}
```

The security layer checks if the user has the appropriate permissions and passes the request on to the actual update
business logic for the blog posts, returning the response. Internally it, of course, needs to fetch the access token
from the database, retreiving the user, possibly involving a caching layer if needed, but that doesn't need to concern
the API.

## Traditional web applications

So far we have only talked about an architecture that is specific to Single Page Applications, where things are nice
and simple. In fact, your application does not have to include *any* state. It can just pass any request that comes in
down the pipeline and the result back up. In essence, your application is basically a collection of functions, often
pure or at least stateless, on steroids with dependency injection. (Shout out to JavaScript folks, we have you to thank
for the rise in functional programming in recent years!)

However, when it comes to traditional web applications, things get messy. They want to store temporary form data and
a bunch of other stuff in *sessions*. While [I don't advocate the use of sessions](/blog/stop-using-php-sessions),
we definitely have to deal with a lot of things an API doesn't have to deal with.

So, here's an idea: why don't we add *one more layer* on top of our API? After all, the permission checking and all
the other things have already been dealt with, so the *web layer* should have to deal with only the things specific
to your traditional web application! 

## TL;DR, the take-away

[Michael Cullum](https://twitter.com/michaelcullumuk) has dubbed this the *Cookie Cutter Approach*, so I'm officially
calling it that. The basic idea is the following:

- Split your application into **Services** and **Entities**.
- Entities should be immutable, and only contain code for validation and creating a changed copy of itself.
- Services should have no internal state, apart from dependencies that are injected.
- Services should have a very low number of public methods, ideally only one, often a *pure* or at least a *stateless*
  function. (Add private methods for readability if you need to, but it is usually better to split the whole class.)
- Services should deal with as little as possible at once. Try to keep them below ~150 lines of code.
- Services should be grouped into *layers*, each layer being responsible for one group of tasks.

Bonus fact: this is not specific to my custom, crazy-ass framework. You can implement this in *any* modern web framework
that supports dependency injection. You just have to be willing to forgo having the framework being used in all parts
of your application.

## The benefits

As you may have realized, this architecture requires you to write quite a lot of code, especially initially. It is not
suited in the least for anything resembling a rapid prototyping approach.

Admittedly, I work on applications that have a very long maintenance period, and customers regularly come with change
requests. Your situation may be different, maybe you make websites that you hand off to the customer who you never
see again, but let me ask you this: when was the last time you took a shortcut and it came back to haunt you?

To me that is one of the most dreaded feelings, seeing the customer come with a relatively simple change request, which
then results in a multiple week headache for the whole team.

This architecture has proven itself to be *consistent*. Not fast, consistent. We know how much time we need to develop
a certain feature. There are no surprises in the system, but it comes with the drawback that we have to write a lot of
the code ourselves.

Additionally, since this setup depends on nothing too specific to Java, I have managed to drop in a semi-experienced PHP
developer and, with the help of an IDE, have them deliver production-ready code in about 3 days.

Also, since everything is so nice and cut up, it is incredibly easy to implement unit testing for the individual parts.

To sum it up, together with my approach to [directory structures](/blog/structure-based-on-intent) it makes quite a 
comfortable system to maintain. 

*I have to say a massive thank you to [Michael Cullum](https://twitter.com/michaelcullumuk),
[Steve Poole](https://twitter.com/spoole167), [Gil Tayar](https://twitter.com/giltayar), Gabor Vereb,
[Ádám Turcsán](https://twitter.com/adam_turcsan), [Goran Spasojevic](https://twitter.com/gogospaso) and
[Dan Radenkovic](http://www.radenkovic.org/) for their input, feedback and ideas that made this architecture what it is
today. (Note that they did not endorse this architecture, but provided input.)*  
 