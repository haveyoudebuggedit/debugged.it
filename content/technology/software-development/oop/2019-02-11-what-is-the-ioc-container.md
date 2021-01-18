---
slug: what-is-the-ioc-container
guid: https://pasztor.at/blog/what-is-the-ioc-container/
authors:
- janos
categories:
- Technology
date: "2019-02-11T00:00:00Z"
publishDate: "2019-02-11T00:00:00Z"
summary: Just what in the name of the allmighy bit is an IoC / Dependency Injection
  container? If find that this confuses a lot of people. Let's dig into it.
images:
- posts/what-is-the-ioc-container.png
preview: posts/what-is-the-ioc-container.jpg
tags:
- Software Development
title: What the *** is an IoC container?
---

{{% tip %}}
**Note:** The code examples in this article are ficticious! The actual method calls depend on your IoC container
implementation.
{{% /tip %}}

## Recap: What is Dependency Injection

Let's face it: software hasn't gotten simpler in the last decades. We get ever more modules, third party libraries, and
in order to manage this concert of dependencies, and not lose our sanity, we have to handle these dependencies somehow.
I have previously written about [dependency handling in depth](/blog/clean-code-dependencies), but let's recap.

Let's say we have two classes: `A` and `B`. Class `A` depends on class `B`. Traditionally you could do something like this:

```java
class A {
  public void foo() {
    B b = new B();
    b.doSomething();
  }
}
```

Here's the issue: if `B` depends on `C` and `C` depends on `D` the whole function becomes really messy. There are other
problems too, but this is the problem that prevents most unexperienced programmers from creating smaller classes and 
leads to spaghetti code.

There is a simple way to restructure this: ask for the dependencies in the constructor, like this:

```java
class A {
  private B b;

  public A(
    B b
  ) {
    this.b = b;
  }
  
  public void foo() {
    b.doSomething();
  }
}
```

This way you request the dependency in the constructor. However, this presents you with a different problem. At some
point in your code you need to do this:

```java
A a = new A(
  new B(
    new C(
      new D()
    )
  )
);
```

So you didn't solve the problem, you just moved it. It's still better than nothing, and this can be moved into a
factory:

```java
class AFactory {
  public A make() {
    return new A(
      new B(
        new C(
          new D()
        )
      )
    );
  }
}
```

However, this is clearly a hassle.

## Enter: the dependency injector

What if we had a tool that kept track of these dependencies and created the instances of `A`, `B`, etc. automatically?

So, this would be the syntax:

```java
A a = injector.make(A.class);
```

How do we do that? That is what the IoC (Inversion of Control) container, or with a different name, the dependency
injector is responsible for. How does this work?

In languages, like Java, the parameters have fixed types. The injector can, for example, use reflection to detect the
type of the dependencies and automatically instantiate the dependencies. In other languages, however, the dependency
tree needs to be manually specified.

## Handling interfaces

Now, you may be asking, what happens if `B` is an `interface`? After all, interfaces and abstract classes cannot be
instantiated. In this case you can configure the injector with an alias, or bind an implementation. This could be done
as follows:

```java
injector.alias(
    B.class,
    BImpl.class
);
```

In this case the injector will substitute all places where `B` is required with `BImpl`. This, of course, assumes that
`BImpl` actually implements the `B` interface. The same goes for abstract classes.

## Factories

Sometimes it is hard to instantiate a class. This is especially true when a third party library needs to be 
incorporated. In this case you can use a factory:

```java
class FooFactory implements Provider<Foo> {
  public Foo get() {
    return Foo.create(
      param1,
      param2
    );
  }
}
```

And then make the injector use this factory:

```java
injector.factory(
  Foo.class,
  new FooFactory()
);
```

## How to use an IoC container?

OK, so you now have this magic thing that can make you anything, any class. Where should you use it? Just anywhere?

No. Your application code, your business logic, should NEVER depend on the IoC container. In fact, your application
should have no concept of the IoC container. The container is the wiring layer of your application. It should be
around the edges of your system.

In [one of my hobby projects](https://github.com/opsbears/webcomponents) I chose to create separate packages for the
implementation and the wiring. The implementation can be entirely used without any of the wiring, if you so chose.  

## Is it magic?

You may be asking: isn't the IoC container magic? Especially if the automatic dependency detection is used?

The answer is: yes, you are right. The IoC container is a little bit of magic. That's the reason why it should never
be part of your core application.

Some people have argued that factories are better, and the IoC container is not necessary. That is certainly an argument
that can be made, but having to write more code also means that people will be inclined to put more code into the same
class. The reason for that is that creating a new class will also mean writing a new factory, which is more code that 
needs writing.

In my eyes moving away from massive monster classes and towards smaller, more compartmentalized classes is definitely
a benefit, and the IoC container makes that possible.