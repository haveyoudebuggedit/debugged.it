---
slug: oop-basics
guid: https://pasztor.at/blog/oop-basics/
authors:
- janos
categories:
- Technology
date: "2016-11-04T00:00:00Z"
publishDate: "2016-11-04T00:00:00Z"
summary: So you've been programming for a while, and you are still stumped with what
  object-oriented programming actually is? Then this may be the guide for you. We'll
  take a departure from traditional explanations and look at a new way of explaining
  OOP.
tags:
- Software Development
title: Getting started in Object-Oriented Programming
---

So you've been programming for a while, and you are still stumped with what object-oriented programming actually is? 
Then this may be the guide for you. We'll take a departure from traditional explanations and look at a new way of 
explaining OOP.

We'll start with a little bit of code right away. Keep in mind that the examples in this article are written in a 
Java-esque notation, but everything can be easily applied to any OOP programming language, whether it is PHP, Python 
or Go.

So, let's look at a class:

```java
class Student {
  string name;
}
```

A class is like a blueprint. If you want to take this blueprint and create an actual *instance*, this is how you do it:

```java
Student myStudent = new Student();
myStudent.name = "Janos";
```

The `myStudent` variable now contains a copy of the `Student` class with the `name` variable set to my name. You 
could also create a second copy just as easily:

```java
Student myStudent = new Student();
myStudent.name = "Janos";

Student yourStudent = new Student();
yourStudent.name = "Your name";
```

If you run this code, the two instances are going to live in separate memory spaces; you can modify them 
independently.

So far so good? All right, then let's do something a bit more advanced. What we have done until now is create a data
structure. You can add as many variables as you like, but it's not much more than a way to organize data. Let's change
that by adding *methods* to our class.

```java
class Student {
  string name;
  void setName(string name) {
    this.name = name;
  }
}
```

As you can see, the `setName` method sets the name variable. Why would you want to do this you ask? Let's imagine a 
situation where you would want to check if the name is empty or not:

```java
class Student {
  string name;
  
  void setName(string name) {
    if (name == "") {
      throw new InvalidArgumentException("The name must not be empty!");
    }
    this.name = name;
  }
}
```

That's quite nice, but as you can see the `name` field can still be unset. We need some method to enforce 
initialization of the `name` field. That's where the *constructor* comes into play.

The constructor is a special function which is executed when the class is instantiated. Usually, it shares the name 
with the class, but in some languages (like PHP) it has a different name. Creating a constructor is just as simple as
creating a method:

```java
class Student {
  string name;
  
  Student(string name) {
    this.setName(name);
  }
  
  void setName(string name) {
    if (name == "") {
      throw new InvalidArgumentException("The name must not be empty!");
    }
    this.name = name;
  }
}

Student myStudent = new Student("Janos");
```

In other words, the parameters you enter at the instance creation automatically end up as parameters of the constructor.

## Defend the data! (Encapsulation)

So as you can see, we bind the data to the functionality. A good class will let you use it without having to know how
it's implemented or how it stores the data internally. However, we have a slight problem. With the current setup we 
can easily bypass the validation logic by setting the `name` variable directly:

```java
Student myStudent = new Student("Janos");
myStudent.name    = "";
```

Fortunately, most OOP languages give us tools to disable external access to member variables and methods, called 
*visibility keywords*. Usually, we distinguish these levels of visibility:

* **public:** Everyone can access the method marked with this keywords. In most OOP languages this is the default.
* **protected:** Only child classes can access the method or variable (we'll talk about this in a bit).
* **private:** Only the current class can access the method or variable. (Be careful, other instances of the same class
can usually access it too!)

With these keywords we can make our code a bit more secure:

```java
class Student {
  private string name;
  
  public void setName(string name) {
    if (name == "") {
      throw new InvalidArgumentException("The name must not be empty!");
    }
    this.name = name;
  }
}
```

If we now try to access the `name` member variable directly, we'll get an error from our compiler. Besides that, 
having explicit visibility markup makes our code a bit more descriptive and easier to read.

As a general rule, you should never use a class as a function container; it should be used to hold the data relevant 
to the class.

## Cooperation of classes

To make OOP useful, you will need more than one class. If you are writing a site displaying articles like this, you 
may want to have a class that can fetch raw data from the database. You would also want a class that transforms 
the raw data, for example, markdown or LaTeX, to the output format, for example, HTML or PDF.

As a naive approach we could do something like this:

```java
class HTMLOutputConverter {
  private MySQLArticleDatabaseConnector db;
  
  public HTMLOutputConverter() {
    this.db = new MySQLArticleDatabaseConnector();
  }
}
```

As you can see, `HTMLOutputConverter` depends on `MySQLArticleDatabaseConnector` and we create an instance of the
database connector in the constructor of the output converter. Why is this bad?

1. You can't replace `MySQLArticleDatabaseConnector` with a different class.
2. Since the `MySQLArticleDatabaseConnector` instance is created in `HTMLOutputConverter`, that class needs to know all
parameters that need to be passed to `MySQLArticleDatabaseConnector`.
3. When looking at the class definition, the dependency isn't immediately apparent. You have to look at the code to find
out there's a dependency.

Let's see if we can do any better. Instead of creating an instance of `MySQLArticleDatabaseConnector`, let's instead 
request it as a parameter. Something like this:

```java
class HTMLOutputConverter {
  private MySQLArticleDatabaseConnector db;
  
  public HTMLOutputConverter(MySQLArticleDatabaseConnector db) {
    this.db = db;
  }
}
```

This construct is called *dependency injection*. The class you depend on is injected, rather than created directly. 
Having to create dependencies manually seems cumbersome at first, but there are tools to help you with that; they are
called *dependency injection containers*. Notable examples include [Google Guice](https://github.com/google/guice),
[Auryn](https://github.com/rdlowrey/auryn) and [Python DIC](https://pypi.python.org/pypi/dic/1.2.0b1).

Dependency injection solves problem number two and three, but not number one. So let's create a new 
language construct and call it an *interface*. An interface would describe the methods a class would need to 
implement, without actually specifying the code for them. Something like this:

```java
interface ArticleDatabaseConnector {
  public Article getArticleBySlug(string slug);
}
```

So an interface would describe a functionality that a class can *implement*. In this case, we would describe a method
 that takes a `slug` parameter and returns an `Article` as a response. You could write the implementing class like this:

```java
class MySQLArticleDatabaseConnector implements ArticleDatabaseConnector {
  public Article getArticleBySlug(string slug) {
    //Query data from the MySQL database and return the article object.
  }
}
```

As you can see, `MySQLArticleDatabaseConnector` implements `ArticleDatabaseConnector` and behaves as required. This 
enables us to modify the `HTMLOutputConverter` to depend on the interface, rather than the actual implementation:

```java
class HTMLOutputConverter {
  private ArticleDatabaseConnector db;
  
  public HTMLOutputConverter(ArticleDatabaseConnector db) {
    this.db = db;
  }
}
```

Since the `HTMLOutputConverter` depends on an interface, we are free to create any implementation to this interface 
we like, whether it is for MySQL, Cassandra or Google Cloud SQL. When one abstraction can have many forms, many actual
realizations, we call that *polymorphism*.

When we use polymorphism in such a way that your classes depend on the abstract, rather than the concrete, we also refer
to that as *dependency inversion* to highlight the fact that we have inverted the dependency.

But that's just fancy geek-speak for the fact that you shouldn't weld your classes together. You can imagine the
interface as being a *contract* between the classes implementing it and the ones using it. This contract describes the
functionality the implementing class must provide, and the using class can rely on to be present.

## Let's generalize interfaces! (Abstraction)

As you might have guessed, interfaces are not the only tools to create an abstraction. As a matter of fact, interfaces
were invented merely as a workaround in Java for something called *multiple inheritance*. Which brings us to the obvious
question: what is inheritance?

Let's imagine a situation when mandating a particular behavior is not enough, you really want to pass some code to your
existing abstraction. Your abstraction would have to give you the possibility to implement some methods, which
interfaces clearly don't do.

That's where inheritance comes into play. Inheritance means, in essence, that if a class `Foo` extends class `Bar`, it
will inherit all of its methods and variables. In other words, you could write code like this:

```java
class Bar {
  protected string baz;
}

class Foo extends Bar {
  public void setBaz(string baz) {
    this.baz = baz;
  }
}
```

As you can see, the child class declares a method that sets a variable inherited from the parent class, which is
possible because the visibility rules (`protected`) allow for it. If we set the variable `baz` to `private`, this code
wouldn't work.

The interesting thing is that in the example above you can instantiate both `Bar` and `Foo`. If you wanted to restrict
that, you would have to declare the `Bar` class `abstract`. In addition to that, you could also add abstract methods
that have no body and must be implemented by child classes:

```java
abstract class Bar {
  protected string baz;
  
  abstract void setBaz(string baz);
}

class Foo extends Bar {
  public void setBaz(string baz) {
    this.baz = baz;
  }
}
```

So how many parents can a class have? One? Two? Five? The answer is: it depends. Some languages, like C++, have solved
the problem of multiple inheritance. So much so that interface language construct does not even exist in C++. Other
languages, like Java or PHP, decided not to deal with this problem and invent interfaces instead. In other words,
interfaces are nothing else than abstract classes with only abstract methods and no variables to circumvent having to
solve multiple inheritance.

{{% warning %}}
**Beware of faulty abstractions!** Many OOP tutorials bring the example of a square inheriting from the rectangle. This is only true in the mathematical sense. In programming you want your child classes to behave the same as their parent classes, which is being violated since the rectangle has two independent edges, where as the square doesn't.
{{% /warning %}}

## Avoiding global state

Some languages, like Java, introduce a special keyword called `static`. It reverses the fact that each instance has its
own memory space and creates a shared memory space across all instances. There are many ways to use it, and even more to
shoot yourself in the foot.

One notable example would be the singleton pattern:

```java
class DatabaseConnection {
  private static DatabaseConnection instance;

  public static DatabaseConnection getInstance() {
    if (!self::instance) {
      self::instance = new DatabaseConnection();
    }
    return self::instance;
  }
}

DatabaseConnection db = DatabaseConnection::getInstance();
```

The first time you call `getInstance`, an instance will be created. Any further calls will return that initial instance.

The problem with using static is that it creates a, sometimes hidden, global state. You cannot create a truly
independent instance of the class you have, which makes testing and some other operations tricky.

In general, you should avoid global state as much as possible. While static is not the only way to create a global
state, it is one of the most pertinent. It is best to avoid using static if at all possible, and going with dependency
injection as described above.

{{% tip %}}
**Tip:** `static` does have some legitimate uses, but in general the alternative should always be considered.
{{% /tip %}}

## Class responsibilities

Having an object that has a state is a departure from the classic, function based programming where you just pass data
around. When learning OOP, try to avoid making your objects pure function containers and integrate data with
functionality.

However, when creating your classes, always consider its responsibility. While it is tempting to put everything related
to one task into said class, it may not be wise to do so. If you have a student management software, you may be tempted
to do something like this:

```java
class Student {
  private string id;
  private string name;
  
  public void setId(string id) { ... }
  public void setName(string name) { ... }
  
  public void save() { ... }
}
```

As you can see in this scenario handling student data and saving it to some sort of database would be in the same class.
In reality, these are two completely separate tasks and don't have any business being in the same. While we won't go
into too much detail in this article, it is recommended that you keep your classes concise and focused on a single task.

## Future steps

These are only the most basic OOP concepts. In reality, there are a multitude of ideas and design patterns that one can
follow, but few programmers can name them by heart. Don't be afraid to experiment, and more importantly, don't be afraid
to fail. OOP, and writing maintainable code in general, is hard so you may need a few tries until you are satisfied with
the results. In future articles, we will elaborate more on the concepts and ideas that can help you write better, more
maintainable code so be sure to stay tuned.
