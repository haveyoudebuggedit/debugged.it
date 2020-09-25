---
slug: why-immutability-matters
guid: https://pasztor.at/blog/why-immutability-matters/
authors:
- janos
categories:
- Technology
date: "2018-11-11T00:00:00Z"
publishDate: "2018-11-11T00:00:00Z"
summary: I've talked about immutable objects in clean code before, but what exactly are they? Why do we use them?
images:
- posts/why-immutability-matters.png
preview: posts/why-immutability-matters.jpg
tags:
- Software Development
title: Why Immutability Matters
---

Immutable objects are an immensely powerful programming concept to avoid all sorts of concurrency issues and a whole
host of bugs, but they are not necessarily easy to understand. Let's take a look at what they are and how we use them.

First of all, let's take a look at a simple object:

```java
class Person {
    public String name;
    
    public Person(
        String name
    ) {
        this.name = name;
    }
}
```

As you can see the `Person` object takes one parameter in its constructor and then puts it into a public `name`
variable. This means that we can do things like this:

```java
Person p = new Person("John");
p.name = "Jane";
```

Simple, right? We can read or modify the data at any given time as we please. However, there are a couple of problems
with this method. First and foremost we use the `name` variable in our class, which means that we irrevocably make the
internal storage of the class part of the public API. In other words, we can never change how the name is stored inside
the class without also having to rewrite large parts of our application.

Some languages provide an ability to install a &ldquo;getter&rdquo; function to work around this (such as C#), but in
most OOP languages you have to do it explicitly:

```java
class Person {
    private String name;
    
    public Person(
        String name
    ) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
}
```

So far so good. If you now wanted to change the internal storage of the name to, say, first name and last name, you
could do it like this:

```java
class Person {
    private String firstName;
    private String lastName;
    
    public Person(
        String firstName,
        String lastName
    ) {
        this.firstName = firstName;
        this.lastName = lastName;
    }
    
    public String getName() {
        return firstName + " " + lastName;
    }
}
```

Setting aside [the massive problems this representation of names brings with it](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/),
you can see that the `getName()` API didn't change.

Now, how about setting names? How about we add something to not only get the name, but also set a name like this?

```java
class Person {
    private String name;
    
    //...
    
    public void setName(String name) {
        this.name = name;
    }
    
    //...
}
```

On its face this looks great because we can now change the name again. However, there's something fundamentally wrong
with changing data like this. There are two reasons: one philosophical and one practical.

Let's look at the philosophical first. The `Person` object is intended to *represent* a person. People do change names,
but then the function should rather be named `changeName` to imply that we are actually changing the name of the same
person, and it should also include the business process of changing the name of the person, not just simply act as a 
setter. `setName` easily leads someone to the conclusion that they can just willy-nilly change the name stored in a
person object with impunity.

The second reason is practical in nature: mutable state (stored data that can be changed) leads to potential bugs.
Let's take this `Person` object and let's define a `PersonStorage` interface as follows:

```java
interface PersonStorage {
    public void store(Person person);
    public Person getByName(String name);
}
```

Note that this `PersonStorage` does not specify *how* that object is stored: in memory, on the disk, or in a database.
The interface also does not force the implementation to *create a copy* of the object it stores. Therefore interesting
bugs can happen:

```java
Person p = new Person("John");
myPersonStorage.store(p);
p.setName("Jane");
myPersonStorage.store(p);
```

Now, how many persons does the person storage store? One or two? Also, if you now use the `getByName` method, which one
will it return?

You see, there are two cases: either the `PersonStorage` makes a copy of the `Person` object, in which case there will
be two `Person` records stored, or it doesn't and simply stores a reference to the object passed, in which case only one
object will be stored with the name &ldquo;Jane&rdquo;. An implementation for the latter could look like this:

```java
class InMemoryPersonStorage implements PersonStorage {
    private Set<Person> persons = new HashSet<>();

    public void store(Person person) {
        this.persons.add(person);
    }
}
```

What's worse, we can even change the data stored without even calling the `store` function. Because the storage only
stores a reference to the original object, changing the name will also change the stored version:

```java
Person p = new Person("John");
myPersonStorage.store(p);
p.setName("Jane");
```

So in essence, the fact that we have **mutable state** lead to bugs in our program. Sure enough, you can work around 
these by explicitly putting the work of creating a copy on the storage, but there is a much easier way: **immutable
objects***. Let's take a look at an example:

```java
class Person {
    private String name;
    
    public Person(
        String name
    ) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    public Person withName(String name) {
        return new Person(name);
    }
}
```

As you can see, instead of the `setName` method we now use a `withName` method which creates a new copy of the Person
object. Always creating a new copy works around the issue of having mutable state. Sure enough, this can lead to some
overhead, but modern compilers can work around that and if you run into performance issues, you can fix those later.

{{% tip %}}
**Remember:** <q>Premature optimization is the root of all evil</q> (Donald Knuth)
{{% /tip %}}

Now, you may argue that a persistence layer keeping a reference to a working object is a broken persistence layer, but
that is a real world scenario. Broken code does exist, and immutability is a valuable tool to prevent such mistakes from
happening.

In more complex scenarios, when objects are passed through multiple layers in an application bugs can creep in easily
and immutability prevents those bugs that are related to *state*. These scenarios can include examples like an in-memory
cache, or out of order function calls.

## How immutability helps with parallel processing

Another important area where immutability helps is parallel processing. More specifically, multithreading. In
multithreaded applications multiple code paths run in parallel, but access the same memory area. Let's look at a very
simple piece of code:

```java
if (p.getName().equals("John")) {
    p.setName(p.getName() + "Doe");
}
```

On it's face this code would have no bugs, but if you run it in parallel, the preemption can mess things up. Let's
look at the above code example with the threaded parts commented in:

```java
if (p.getName().equals("John")) {

    //The other thread changes the name here, so it is no longer John
    
    p.setName(p.getName() + "Doe");
}
```

This is a race condition. The first thread checks if the name is &ldquo;John&rdquo;, but then the second thread changes
the name. The first thread still proceeds under the assumption that the name is John.

You could, of course, employ locking to ensure that only one thread enters the critical section at any given time, but
that can be a bottle neck. However, if the objects are immutable, this scenario can't happen since the object stored in
`p` is always the same. If the other thread wants to affect a change, it creates a new copy, which will not be present
in the first thread.

## Summary

In general, I would recommend making sure that you have *mutable state* in as few places as possible in your 
application, and even when you do, tightly gate it with properly designed APIs and not let it leak into other parts. The
less code parts have state to them, the less likely it is that state-related errors crop up.

Obviously you can't program entirely without state in most programming tasks, but thinking about data structures
as immutable by default makes your program a lot more resilient towards random bugs. **When you really need to introduce
mutability, you will then be forced to think about the implications instead of having mutability all over the place.** 