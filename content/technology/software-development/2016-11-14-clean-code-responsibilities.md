---
slug: clean-code-responsibilities
guid: https://pasztor.at/blog/clean-code-responsibilities/
authors:
- janos
categories:
- Technology
date: "2016-11-14T00:00:00Z"
publishDate: "2016-11-14T00:00:00Z"
summary: I heard you want to be a better coder. You want to use reusable pieces, and
  you want to have an easier time maintaining older code. You may also want to work
  better in a team and ensure there are less bugs.
tags:
- Software Development
title: 'Clean Code: Responsibilities'
---

The desire for better code usually leads people to discover the term “clean code”. It was most probably coined by 
[Robert C. “Uncle Bob” Martin](https://cleancoders.com/), who wrote
a book with the same title. You might want to give it a read, although, I find it to be
very wordy. The book covers a few underlying principles that should help you write modular code in such a way that you
can later reuse those modules. In this series we are going to cover his principles and ideas, as well as those of 
some other authors of the clean code movement.

You might have noticed, I used the name “module” and not “class” or “object”. That's because clean code is not 
specific to object-oriented programming. You can use clean code principles with any programming paradigm you 
like, for example, classic procedural programming.

So let's get started with the first topic on our agenda: when is a good time to split up your modules? 

## Responsibilities

When writing your code, you have to segment it somehow. The unit of organization can be classes or 
modules, depending on your programming paradigm. The general idea is that one piece of code should only deal with **a 
single responsibility**. In other words, do one thing and do it well.

But what is a responsibility? Let's take a class that, for example, deals with students of a school. This class 
will hold the data for exactly one student. Just like this:

```java
class Student {
  private string name;
  
  public void setName(string name) {
    this.name = name;
  }
  
  public string getName() {
    return this.name;
  }
}
```

Obviously, you need to save the data somewhere, for example, in a database. The question presents itself: is it a 
good idea to implement a `save()` method that stores this student record? After all, this would be terribly convenient:

```java
Student joe = new Student();
joe.setName('Joe');
joe.save();
```

Easy, right? Well, not so fast. Imagine the following situation. You implement this class with MySQL in mind, which 
is a fairly standard database engine in the web world. One faithful day, your boss comes to you and tells you that the 
system administrators have been complaining, the servers are overloaded. After a brief hunt you discover that your 
`students` table is insanely large and slow, so you now decide to implement caching for your student data. The data is 
read from MySQL and written, for example, into Memcache.

So now your Student class needs to know about both MySQL and Memcache. By now your simple `Student` class that was 
only supposed to give you easy access to the student data has grown to a considerable size and now presents a 
maintenance problem. There's a lot of code which you can't even test. But hey, such is life, right?

The following week, déjà vu, your boss is at your desk again. The sysadmins are complaining again. (Can't they just 
buy more hardware? Come on.) Now it's your `courses` table that's causing problems. You decide to go the same route 
and copy over the code for Memcache to your `Courses` class.

Yes, yes, I can hear you scream that you would never do that. You would always decide to refactor your code to avoid 
duplication. But believe me, others won't. Unless you work alone, you will have to deal with people who have a higher 
tolerance for duplicated code.

So how can we avoid this situation? How can we make sure this doesn't happen, even if it isn't you editing the code? 
The answer lies in the word *responsibility*. With our `save()` method we have made a mistake. We have put more than 
one responsibility into one class. The `Student` class is responsible for both holding the student record and saving 
it to the database.

In the words of the great Uncle Bob, **a class has a single responsibility if it has one and only one reason for 
change**. The previous example is clear: there are two responsibilities, storing/retrieving the data and the data 
structure itself. These should be decoupled so we can change them independently.

So how do we fix this? Remember, we said that we wanted to have one class to have only one responsibility. So let's 
split it in two. Let's keep the original `Student` class as it is and create a second class for storing and 
retrieving the `Student` object.

```java
class MySQLStudentStorage {
  public Student getById(int id) {
    //...
  }
  
  public void store(Student Student) {
    //...
  }
}
```

Of course, this is easy when you have practically no code yet. But what do you do if you already have a ton of code 
that relies on your save() function? Well, you're in a tough situation, and there is no perfect solution.

You could, for example, proxy through calls from the `Student` class like this:

```java
class Student {
  //...
  
  public void save() {
    storage = new MySQLStudentStorage();
    storage.store(this);
  }
}
```

Using this proxy solution will help you greatly because you can rewrite your code one module at a time and you don't 
need to push a huge change to your production environment. The proxy strategy can be used in almost all 
situations when you need to split up a class.

## Refactoring

Let's look at a different example. Imagine a system that handles financial data. You know, the boring stuff. You will
most probably have some sort of database, and you will have to provide some reports to the people working with it. 
These reports will be either Excel tables or CSV files of some description.

One such report could be the monthly income/expense sheet. Initially you built this sheet for the finance people so 
they can run their fancy reporting software on the data you provide. One day, much to your surprise, the CEO comes 
into your room and asks you to change one column of the report. She wants you to format the numbers nicely so it's 
easier for her to read. Naturally you want to keep your CEO happy, so you oblige and change that one column.

All's well, until at the end of the month the finance people try to use the report again and import it into their 
software. All hell breaks loose. They want the report fixed. I don't know if you have met finance people, but if they
say they want their report fixed, it means they want it fixed *now*. After all, it's the end of the month and they 
need to deliver their own reports and nobody will give a damn if they are late because of you.

What do you do? Obviously, you revert the change. Then you write an apologetic letter to the CEO, which 
definitely won't look good on your score card the next time you want a raise. Or you could just take the time and 
refactor your code to provide *two* separate reports instead of one and sell it to the CEO as a feature. Needless to 
say, you have to spend some time on it, but having a dedicated CEO report is much better than having a black mark on 
your record. Also, the next time the CEO asks you to change something, you will have the freedom to do so because you
have made sure that one module is only responsible for one thing, only has *one reason for change*. 

## Conclusion

As you can see, the **single responsibility principle** outlined in this article very much aligns with business 
interests. It is in the long term interest of a company to have you writing code that follows this idea. Squeezing in 
the change into a class where it doesn't belong will result in larger and larger classes and cause huge maintenance
headaches. Having convoluted code also means that any change you need to make will take months instead of days, because
you have to untangle the mess that has *organically evolved* in your code. 

## Up next

Awesome! So we have our responsibilities neatly split up, every class or module has only one thing to do. But what 
happens if you need to replace a module? Replacing class names gets old pretty fast, so we need a better solution. 
But that's a topic for another day.