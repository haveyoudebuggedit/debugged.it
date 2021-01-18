---
slug: oop-misunderstandings
guid: https://pasztor.at/blog/oop-misunderstandings/
authors:
- janos
categories:
- Technology
date: "2019-01-05T00:00:00Z"
publishDate: "2019-01-05T00:00:00Z"
summary: Is OOP dead? Is Functional Programming the future? Some articles seem to suggest so. I would tend to disagree. Let's discuss!
images:
- posts/oop-misunderstandings.png
preview: posts/oop-misunderstandings.jpg
tags:
- Software Development
title: What people misunderstand about OOP
---

Every few months I come across a blog post where the author brings up seemingly legitimate issues with object-oriented
programming and declares that it is a thing of the past, we should all move to functional programming instead.

As I discussed before, I made [an argument that OOP and FP are not contradictory](/blog/functional-object-oriented-programming).
In fact, I managed to use them in conjunction with great success.

Why is it that the authors of these articles have so many issues with OOP and why does FP seem to be such an obvious choice?

## How OOP is taught

When OOP is taught in school, it is usually taught as being held up by these four principles: *Encapsulation*,
*Inheritance*, *Abstraction*, and *Polymorphism*. This is usually also the list of items that articles, discussing the 
demise of OOP, tend to attack.

OOP, like FP, however, is a tool. It is a tool to do a job. As such, it can be used and abused. If you, for example, 
create a false abstraction, you are abusing the tool.

The `Square` class, for example, should *never* extend the  `Rectangle` class. In a mathematical sense they are, of
course, related. In a programming sense, however, they are not in an inheritance relationship. The reason is that the
requirements are tightened when it comes to squares. While rectangles can have two independent side lengths, squares
have a strict requirements that the sides all be equal. 

## Inheritance

Let's talk about inheritance a little more. Inheritance is often described as the A and O of OOP. It is common to see
textbook examples where beautiful hierarchies of inherited classes are built to solve a problem. However, in practice
you rarely use inheritance. Instead, *composition* is often used.

Let's look at an example. Let's say we have a very simple class, a *controller* in a web application. Most modern 
frameworks would ask you to do something like this:

```java
class BlogController extends FrameworkAbstractController {
}
```

This, presumably, makes it easy to do calls like `this.renderTemplate(...)` because they are inherited from the 
`FrameworkAbstractController`.

As many of the articles in question point out, this presents some very valid problems. Any internal function of the
base class is now actually an API. It can no longer change. Any protected variables of the base controller will now be
more or less part of the API.

This is incredibly easy to mess up. Instead, with composition and dependency injection, you would do this:

```java
class BlogController {
    public BlogController (
        TemplateRenderer templateRenderer
    ) {
    }
}
```

You see, now you don't depend on some nebulous `FrameworkAbstractController` any more, but you depend on a really well
defined and narrow thing, the `TemplateRenderer`.  The `BlogController` actually has no business inheriting anything
from any other controller, because it does not inherit any behaviors.

## Encapsulation

The second, often attacked feature of OOP is encapsulation. In good English encapsulation means that the data and
functionality are delivered together and the internal state of a class is hidden from the outside world.

This, again, can be used and abused. A prime example of abuse is leaky state.

For the sake of argument let's say that the `List<>` class contains a list of elements and this list can be modified.
Let's create a shopping cart handling class as follows:

```java
class ShoppingCart {
    private List<ShoppingCartItem> items;
    
    public List<ShoppingCartItem> getItems() {
        return this.items;
    }
}
```

What happens here in most modern OOP languages is that the `items` variable will be returned by reference. So I can then
do the following:

```java
shoppingCart.getItems().clear();
```

And this will effectively clear the list of items in the shopping cart, without the `ShoppingCart` even knowing about
it. If you pay close attention, however, this isn't even a fault of the encapsulation principle. This is a *violation*
of that principle, because the `ShoppingCart` class *leaks* the internal state.

In this specific example the author of the `ShoppingCart` class could use [immutability](/blog/why-immutability-matters)
to work around the problem and ensure that the encapsulation is not violated.

A different way inexperienced programmers often violate encapsulation is by introducing *state* where none is required.
Often inexperienced programmers use private class variables to pass data from one function to another within the same
class instead of using Data Transfer Objects to pass a complex structure to a different function. This introduces
unnecessary complexity and often leads to bugs.

In general, it is a good idea to avoid state (storing mutable data) in our classes whenever possible. If we do, it 
should be *well encapsulated* and made sure that it does not leak.

## Abstraction

Abstraction is, again, very much misunderstood. You should in no way pack your code full of abstract classes and make
deep hierarchies.

If you do that without any good reason, you are just asking for trouble. It does not matter if the abstraction is
done as an abstract class or an interface, these introduce additional complexity. This complexity has to be justified.

In laymans terms, you should only create an interface, if you are actually going to take the time to document the 
*behavior* that is expected from an implementing class. Yes, you read me right. Don't just write down the list of
functions that need to be implemented, write down how they are supposed to behave.

## Polymorphism

The last item in our list is polymorphism. It means that one class can implement many behaviors. The bad textbook
example is that a `Square` can be a `Rectangle` as well as a `Parallelogram`. Well, as discussed above, that's in no way
true in OOP as their behaviors differ.

When talking about polymorphism, one should think about *behavior* instead of *code*. A good example would be the
`Soldier` class in a computer game. It could implement both the `Movable` behavior (as in: it can move) and the
`Enemy` behavior (as in: shoots you). In contrast, the `GunEmplacement` could just implement the `Enemy` behavior.

So, just because you can write `Square implements Rectangle, Parallelogram`, that doesn't make it true. Your
abstractions need to actually work in a *business sense*. You need to think about *behavior* more than *code*. 

## Why FP isn't the silver bullet

Now that we've gone through the four principles, what is this Functional Programming thing and why doesn't it solve all
our code problems?

In the eyes of many FP believers, classes are an *abomination unto the lord*, and code should be represented as 
*functions*. Depending on the language, data can be passed betweeen them using primitive types or a structured data
set (arrays, maps, etc).

Additionally, the majority of functions should not have *side effects*. In other words they should not modify some data
in some other place in the background, but only work on the input parameters to produce the output.

This approach separates the *data* from the *functionality*, which is at a first glance fundamentally different from
the OOP approach. Its draw is that it keeps things *simple*. You want to do something, you write a function, end of
story.

The problem comes in when functions need to rely on each other. When function `A` calls function `B` and function `B` 
calls half a dozen other functions, and at the end of the chain there is a
[left-pad](https://www.theregister.co.uk/2016/03/23/npm_left_pad_chaos/) function that can suddenly break, you have a
problem.

Most programmers who consider themselves FP, love FP for its simplicity and don't think of this as a big problem.
Which is fair enough, if you just have to ship the software and never touch it again. However, if you want a
maintainable code base, you better adhere to the [clean code principles](/tags/clean-code). That includes using
[dependency inversion](/blog/clean-code-dependencies), which makes FP also a lot more complex to use.

## OOP or FP?

OOP and FP are *tools*. It doesn't matter what programming paradigm you use. The problems presented in most of the 
articles are about *organizing* your code.

To me the *macrostructure* of the application matters a lot more: what are the modules? How do they communicate with 
each other? What are the common data structures? How are these documented? What are the important business objects?

These are all questions that have nothing to do with the programming paradigm in use, and the programming paradigm
doesn't even solve. A good programmer will learn the paradigm as a matter of knowing ones tools, and will use which
ever is appropriate for the given task.