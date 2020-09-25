---
slug: functional-object-oriented-programming
guid: https://pasztor.at/blog/functional-object-oriented-programming/
authors:
- janos
categories:
- Technology
date: "2017-10-19T00:00:00Z"
publishDate: "2017-10-19T00:00:00Z"
summary: Does the title strike you as strange? Do you think functional and object
  oriented programming are two fundamentally contradicting paradigms? I don't think
  so.
images:
- posts/functional-object-oriented-programming.png
preview: posts/functional-object-oriented-programming.png
tags:
- Software Development
title: Functional Object Oriented Programming
---

Here's the thing: I think functional programming is a good thing, but is lacking a few things that I found to be very
good tools. So, I want to marry the good parts from both and create a paradigm that brings
along the best of both worlds. Join me on a trip through paradigm-land!

Before we start into the general idea, let's take a look at the good and the bad from both worlds. Let's take a closer
look at the bride and the groom.

{{% warning %}}
**Bias warning**: this article is written from a standpoint of a web developer who left PHP for Java because of static
typing. Since I have only ever worked in web development, servers, etc. I cannot speak for, say, games development. If
you have any feedback regarding this topic, feel free to [contact me](/imprint/).
{{% /warning %}}

## Object Oriented Programming

Object oriented programming has been around and in mainstream use for a long time. Yet, it took almost two decades until
we figured out how to apply the S.O.L.I.D. principles, how to write code that isn't a nightmare to maintain. Many
books still teach Encapsulation, Inheritance, Polymorphism, and Abstraction as being the alpha and omega of OOP, and
gloss over the importance of good code organization entirely. However, the industry has slowly learned the importance
of loose coupling, dependency injection and all the other goodies that can be achieved with OOP.

If we break them down, most modern OOP programs contain two types of classes: data structures and classes that actually
do something. Wait, don't light your torches yet, let me explain. Usually a data structure will look like this:

```java
class BlogPost {
  private final String title;
  private final String text;
  
  public BlogPost(String title, String text) {
    this.title = tite;
    this.text  = text;
  }
  
  public String getTitle() {
    return title;
  }
  
  public String getText() {
    return text;
  }
}
```

As you can see this class serves no other purpose than to contain the blog post in a structured form. While some 
programmers will still use setters, this variant has already borrowed much from the functional world by being
*immutable*. You cannot change this data structure, you can only create a new one as a copy. This is also often achieved
by a copy constructor.

You may also note that this type of class has no methods that actually do anything. Sure, they may contain various
convenient ways to create them, such as copy constructors from other objects. They may also contain various getters
that help with fetching the data from the object. But they will never ever contain any method to, say, save themselves
to a database. Some badly written ORMs (I'm looking at you Eloquent) insist on doing this, but most people have realized
that having a data structure class save itself often leads to maintenance problems. I won't go into details here, feel
free to look it up if you don't believe me.

Now, the other type of class is the one that actually does something. Let's say you have a controller in a web system,
for example:

```java
class BlogController {
  private final BlogPostBusinessLogic blogPostbl;
  
  public BlogController(BlogPostBusinessLogic blogPostbl) {
    this.blogPostbl = blogPostbl;
  }
  
  public ViewModel handleList() {
    List<BlogPost> blogPosts = blogPostBl.getLastestPosts();
    
    ViewModel viewModel = new ViewModel();
    viewModel.set("posts", blogPosts);
    return viewModel;
  }
}
```

As you can see this class has almost no state. The only thing it has is its dependencies that are injected using the
controller, and possibly some configuration.

## Functional programming

With that in mind, let's take a look at functional programming. Contrary to popular belief, functional programming has
also been here for a *very* long time, but it had much less prominence than OOP. The basic premise in functional is the
total avoidance of *state* wherever possible and with it avoid any side effects. This means that in pure functional
methods variables cannot change their value.

Let's look at an example, the classic Fibonacci numbers. Tradifionally you would write something like this:

```java
int fibonacci(int n) {
  if (n < 3) {
    return 1;
  }
  int firstNumber = 1;
  int secondNumber = 1;
  for (int i = 2; i < n; i++) {
    int tmpFirstNumber = firstNumber;
    firstNumber = firstNumber + secondNumber;
    secondNumber = tmpFirstNumber;
  }
  return firstNumber;
}
```

However, upon closer inspection we can also write this code much simpler:

```java
int fibonacci(int n) {
  if (n < 3) {
    return 1;
  }
  return fibonacci(n-1) + fibonacci(n-2);
}
```

With proper compiler optimization this is as efficient as the non-functional version.

However, as you attentive readers may notice, in a real-world application the application state has to live *somewhere*.
Since functional tries to avoid state as much as possible, nowadays state is often relegated to the outskirts of the
application. One such methodology popular with web developers is [flux](https://facebook.github.io/flux/).

Whenever an action is triggered, say a click of a button, you would normally explicitly change the state of the
application. In flux what happens instead is that a bunch of pure functions are called that are fed the current state
and have to return the changed state. This changed state is then mapped to the UI components.

While in and of itself this is a great idea to deal with the lack of tools in JavaScript, it leaves many things to be
desired, among others the lack of abstractions. Say we have a reducer that needs a base64 function:

```java
function attachmentReducer(state, action) {
  switch (action.type) {
    case ACTION_ATTACHMENT_SET:
      return {...state, attachment: base64_encode(action.attachment_data)};
      break;
    default:
      return state;
  }
}
```

As you can see the attachmentReducer function is now tighly bound to the `base64_encode` function. Unfortunately we have
no simple way to introduce an abstraction, adding a provider for base64 encoding. What's more, said base64_encode
function may be used in hundreds of places, so replacing said external library can involve a major pain in ones
backside. There are ways around that, but as it stands, the web environment doesn't exactly make it easy.

## Bringing both worlds together

Let's think about it: functional is a great idea. It saves us a lot of time by not having to hunt for bugs, being easy
to run in parallel. OOP is great for abstractions, dependency injection and a bunch of other stuff. So why don't we
bring the two together in holy matrimony.

Let's take an OOP language like Java and let's impose the following rules on the core classes:

1. Data structure classes must be populated using the constructor, must be immutable and must have no dependencies apart
   from what the language provides.
2. For non-data structure classes all dependencies and configuration must be injected using the constructor.
3. For non-data structure classes, apart from the constructor, only one public method is allowed.
4. All dependencies must be to interfaces. 
5. All class variables must be declared final.
6. Class variables that are not immutable (say, a HashMap or an ArrayList) must be treated as if they were. No changes
   are allowed.

What do we get? Well, something like this:

```java
class BlogController {
  private final BlogPostBusinessLogic blogPostbl;
  
  public BlogController(BlogPostBusinessLogic blogPostbl) {
    this.blogPostbl = blogPostbl;
  }
  
  public ViewModel handleList() {
    List<BlogPost> blogPosts = blogPostBl.getLastestPosts();
    
    return new ViewModel().with("posts", blogPosts);
  }
}
```

Apart from the injected business logic, this class is entirely *functional* and has no side effects. However, we still
get the benefits that OOP brings with it, such as abstractions, dependency injection, etc.

## Real-world use

I'm not one of those functional nutjobs that think that functional is the only true god and everything else is
blasphemy. State is a necessary evil in programming and we have to live with it. But it doesn't mean that we should put
*all* our code full of state. Instead, we should try and minimize state to the places that truly need it, such as
caching and storage implementations, or wrapper classes for external libraries.

{{% tip %}}
**Follow-up article:** If you wish to learn more about this topic, I recommend reading my follow up article “[One controller, one action](/technology/one-controller-one-action)”.
{{% /tip %}}

I have been using it for a while like this and I felt no adverse effects. What do you think? Would you code like this?
[Let me know!](/imprint/)