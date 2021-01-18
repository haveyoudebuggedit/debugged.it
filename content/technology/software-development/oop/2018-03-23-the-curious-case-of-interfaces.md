---
slug: the-curious-case-of-interfaces
guid: https://pasztor.at/blog/the-curious-case-of-interfaces/
authors:
- janos
categories:
- Technology
date: "2018-03-23T00:00:00Z"
publishDate: "2018-03-23T00:00:00Z"
summary: Interfaces are like internal APIs for your code. So how should you create them?
images:
- posts/interfaces.png
preview: posts/interfaces.jpg
tags:
- Software Development
title: The curious case of Interfaces
---

No matter what kind of object-oriented programming language you are using, you have a way to
declare abstractions. PHP and Java, for example, have a dedicated language construct called
interfaces, other languages like C++ have abstract classes to inherit from. Regardless, this article
will keep things fairly generic so don't worry about the exact language construct.

## What are interfaces?

So why would you even use an interface, especially in a [dynamically typed language](/blog/loose-strict-static) like PHP
or Python? After all, you don't *have* to declare your types, your can simply use methods without declaring
types like this?

```php?start_inline=1
function baz($logger) {
    $logger->log("Hello world!");
}
```

Simple, right? Well, yes. If you are not in a statically typed language like Java, you *can* certainly do this.
However, there is a problem with the code above: how should someone using the `baz()` method know what `$logger` is?
What kind of methods does it need to declare? How should it work?

A trivial answer would be *&ldquo;read the code&rdquo;*, but after about ten to twenty thousand lines of code
remembering everything really becomes a chore. Alternatively, we could also document our functions:

```php?start_inline=1
/**
 * @param $logger needs to have a log() with a single string parameter
 */
function baz($logger) {
    $logger->log("Hello world!");
}
```

However, again, this is not an optimal solution. Every programmer I've seen so far has been more on the lazy
side. When deadlines are looming and the after-work beer is calling, the least of your worries is going to be that
pesky little piece of documentation that you should have updated.

Therefore we need *executable documentation*. Something that the compiler or program itself is going to check
and is going to force us to keep up to date. This tool is called a *static type checker* or *static code analyzer*.

Java, for example, has this built in. Python and other languages have an external type checker (e.g. mypy) that you can
use for this purpose.

Now, if we have this sort of executable documentation, we have an opportunity. Normally you would declare some
implementation like this:

```php?start_inline=1
class Logger {
    function log(string $message) {
        //Write log to file
    }
}
function baz(Logger $logger) {
    $logger->log("Hello world!");
}
```

However, if we don't implement that method right there, we can describe a functionality with it, just like this:

```php?start_inline=1
abstract class Logger {
    abstract function log(string $message);
}

class FileLogger extends Logger {
    function log(string $message) {
        //Write log to file
    }
}
```

Or, if we wanted to have a little bit of syntactic sugar:

```php?start_inline=1
interface Logger {
    function log(string $message);
}
```

> An interface, from a syntactic standpoint, is nothing else than an abstract class with only abstract methods in it. It
> was created to help out in languages that don't have multiple inheritance. However, nowadays it has a much deeper
> meaning. (See below.)

## The purpose of interfaces

Now, as you have seen above, interfaces are powerful tools to *describe functionality*. So, not only do we enforce a
certain method signature, but we can also convey what the method is supposed to do, such as:

```php?start_inline=1
interface Logger {
    /**
     * Log a certain message to the application log.
     * If the log cannot be written, it should throw an exception.
     * 
     * @param string $message A message that can contain any character,
     *                        including line breaks, and can be any length.  
     *
     * @throws LogMessageTooLongException if the log message is too long to be written.
     * @throws LogFailedException if writing the log message failed for whatever reason.
     */
    function log(string $message);
}
``` 

As you can see, the interface not only enforces a certain type of method signature, it also conveys
meaning as to how the implementation has to be have in certain scenarios.

If this description is adhered to, the implementation can be easily replaced by simply passing a
different instance. I've written about this in detail in [Clean Code: Dependencies](/blog/clean-code-dependencies).

But that is not the only purpose for having interfaces. After all, most applications *probably* don't want to have 
multiple implementations for the same functionality. (Although once you get into it, it becomes a really handy tool.)

The purposes of interfaces is the ability to write selfcontained tests. Let's stick with this logger example for a
moment and let's assume we have a class that needs this logger to work. Say, a `BlogPostCreationBusinessLogic`. it looks
somewhat like this:

```php?start_inline=1
class BlogPostCreationBusinessLogic {
    private $blogPostStorage;
    private $logger;
    
    function __construct(BlogPostStorage $blogPostStorage, Logger $logger) {
        $this->blogPostStorage = $blogPostStorage;
        $this->logger = $logger;
    }
    
    function create(string $title, string $body) : BlogPost {
        $blogPost = new BlogPost(
            $title,
            $body
        );
        $blogPost = $this->blogPostStorage->create($blogPost);
        $logger->log("Created new blog post with ID " . $blogPost->getId());
        return $blogPost;
    }
}
```

If we wanted to test this we could go the way of setting up a test database, resetting it after every test and testing
the whole process. This would be very error-prone and would probably lead to a lot of flaky test. Tests that fail
with no apparent reason. Tests like that are bad because they erode the confidence in them.

Instead, we could also write unit tests that isolate this business logic:

```php?start_inline=1
function testBlogPostCreation() {
    $logger = new FakeLogger();
    $storage = new FakeBlogPostStorage();
    
    $bl = new BlogPostCreationBusinessLogic($storage, $logger);
    $blogPost = $bl->create("Hello world!", "What's up?");
    
    $this->assertEquals("Hello world!", $blogPost->getTitle());
    //...
}
```

So in this case `FakeLogger` and `FakeBlogPostStorage` are implementations of their respective interfaces, but instead
of writing things into a real database, they store data internally so they can be queried later. Interfaces
helped us to isolate the business logic from its I/O channels and made tests reliable.

## How many methods in one interface?

Now, here comes the question: how do you design a good interface?

Let's take a look at an example, a payment interface. Most developers tasked with creating an interface
for a business logic part dealing with blog posts, would probably think along the lines of requiring a certain
functionality such as this:

```php?start_inline=1
interface BlogPostBusinessLogic {
    function create(string $title, string $body) : BlogPost;
    function getById(string $id) : BlogPost;
    //...
}
```

However, among the SOLID principles recommended for good, maintainable code there is one called *Interface Segregation
Principle*. It states:

> No client should be forced to depend on methods it does not use.

In other words, the above `BlogPostBusinessLogic` may not be up to snuff and may need to be split into 
multiple interfaces. But why though?

Say, your controller looks like this:

```php?start_inline=1
class BlogPostController {
    private $blogPostBusinessLogic;
    
    function __construct(BlogPostBusinessLogic $blogPostBusinessLogic) {
        $this->blogPostBusinessLogic = $blogPostBusinessLogic;
    }

    function showAction(string $id) {
        $blogPost = $this
            ->blogPostBusinessLogic
            ->getById($id);
        //Do something with $blogPost
    } 
}
```

In this case your controller only depends on the `getById()` method, but the `BlogPostBusinessLogic` interface also
requires that the `create()` method be implemented.

If you are to write a test for this controller now, you need to create a fake implementation like this:

```php?start_inline=1
class FakeBlogPostBusinessLogic implements BlogPostBusinessLogic {
    private $prepopulatedBlogPosts;
    
    function __construct(array $prepopulatedBlogPosts) {
        $this->prepopulatedBlogPosts = $prepopulatedBlogPosts;
    }

    function create(string $title, string $body) : BlogPost {
        //Do nothing?
    }
    function getById(string $id) : BlogPost {
        return $this->prepopulatedBlogPosts[$id];
    }
    //...
}
```

As you can see, we have a dangling `create()` function that has no implementation. Apart from being plain ugly, someone
may be tempted to use this fake implementation later on, only to discover that the create method contains no usable
implementation.

If you split the above interface in two, you can still create one backing class that implements both interfaces, but the
class using them doesn't have to know that it's the same.

In essence, the purpose of interfaces is a form of a contract describing one or more standardized methods and their
functionality between the caller and the callee. Both classes, the one calling the interface and the one receiving the
call, can be replaced without any modification to the other,
[as long as the dependencies are properly handled](/blog/clean-code-dependencies).
