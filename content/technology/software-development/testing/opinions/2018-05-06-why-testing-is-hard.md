---
slug: why-testing-is-hard
guid: https://pasztor.at/blog/why-testing-is-hard/
authors:
- janos
categories:
- Technology
date: "2018-05-06T00:00:00Z"
publishDate: "2018-05-06T00:00:00Z"
summary: Why do so many developers struggle with testing? Why don't we all have 9x% test coverage on our code?
images:
- posts/why-testing-is-hard.png
preview: posts/why-testing-is-hard.jpg
tags:
- Testing
- Software Development
title: Why testing is hard
---

When most people start testing their software, they are driven by one urge: if they user does X, then Y should happen.
In other words, *&ldquo;we broke the damn signup flow too many times, let's make sure it doesn't happen again.&rdquo;*

More often than not, this kind of thinking leads to tools like [Selenium](https://www.seleniumhq.org/),
[Behat](http://behat.org/), etc. And these are great tools, but not for this task. Let's explore why.

## Integration testing *everything*

When we think of a modern web application in a very abstract way, we can draw something like this:

![The user interface depends on the business logic, the business logic depends on the database.](posts/why-testing-is-hard/module-dependencies.svg)

Let's assume you want to create a set of tests that cover *most* possible cases. So **how many test cases do we have to write?**

The number of tests required to fully cover the application requires can be measured by something we call *cyclomatic complexity*, or simply put: how many decision points are there in the code, how many different paths of execution are possible, depending on the input and the database. For each possible execution path we have to write at least one test in order to cover the application.

So, if your whole application contains one `if` case with a simple decision, you need two tests: one if the `if` results in true, and one if the `if` results in false, resulting in a complexity of 2.

However, looking at the graph above, the number of different cases already starts to climb. Think about how many decisions you need to make when processing a single request and what factors that request depends on? For each possible execution path you'd have to have a test case, each test case consisting of setting up the database correctly, running the test and evaluating the results, which is quite a lot of work, and is also very slow to run.

This issue is compounded by the fact that every single change in the user interface requires touching potentially dozens of test cases. Needless to say, that can become a problem really fast when for every design change you are looking at a workload of potentially several hours. (And I'm not even mentioning the problem of accidentally running your tests against the production database... yes I've seen someone make that mistake.)

Long story short, doing *only* integration tests, especially when also putting the user interface with its million
buttons and form fields into the mix, is a sure fire way to fail and stop maintaining the tests pretty soon.

## Introducing: *unit tests*

Ok, so it's pretty clear that we need to split this application somehow and test each component individually, then just test if they are wired together correctly. This method of *only testing one component* is called **unit testing**.

Sounds easy enough, doesn't it? There is but one problem left to solve: how do we split the application? After all, the function calls from the *User Interface* to the *Business Logic* and from the *Business Logic* to the *Database* are hard-wired!

And that's where the real crux of the matter lies: **you can't start writing useful tests if your code is not written in a testable way!**

Somehow we need to *decouple* the UI from the Business Logic and the Business Logic from the Database. To test the
Business Logic, for example, we need to pass in a simplified, *dumbed down* version of a database connector that is easier to set up with test data.

## Refactoring for testability

Let's take a look at one such unit that we want to test. In modern-day web applications one such unit would be the 
*controller*. Let's take the [Symfony](https://symfony.com/) framework as an example. Their
[own documentation](https://symfony.com/doc/current/doctrine.html#persisting-objects-to-the-database) guides the user
down a path where the controller is accessing the database directly.

Applications written like that are extremely hard to test because they directly depend on an actual database being
available. So what can you do?

In our case, let's create a separate class for accessing data:

```php?start_inline=1
class BlogPostStorage {
    public function getLatestBlogPosts($count = 10): array {
        //do database stuff here
    } 
}
```

Sweet! We now have a separate class to deal with SQL-related things, and for the purposes of testing the controller,
we can just pass a *fake* implementation that doesn't event need an actual database. Much faster, much easer to set up.

But how will the controller get an instance of the `BlogPostStorage` class? After all, we need to be able to replace it,
right?

Well, that's where [dependency injection](/blog/clean-code-dependencies) comes into play. Instead of creating a new
copy of the storage directly in the controller, we *ask for it in the constructor*:

```php?start_inline
class BlogPostController {
    public function __construct(
        BlogPostStorage $storage
    ) {
        //...
    }
}
```

This enables us to replace the storage with a different implementation. Even more so if we
[define an interface for it](/blog/the-curious-case-of-interfaces).

{{% warning %}}
**Warning!** Many framework docs lead you down a rabbit hole of bad practices, encouraging you to use patterns like static method calls or service locators, which will make testing hard. Be sure to use dependency injection if you want testability.
{{% /warning %}}

### Getting rid of global state

Another factor in untestable code is the presence of hidden global state. Global state can come in many forms, such as global variables like `$SESSION` in PHP, the use of the singleton pattern, etc.

For the sake of argument let's assume your controller is using `$SESSION`. For your tests you'd set up this variable with a certain value beforehand and then run your tests. Now, imagine a situation where you *forget* to reset the contents of this variable and something, somewhere deep in your controller relies upon that. If you run test `A` before test `B`, everything passes. If you run test `B` first, it fails. In other words your tests become *flaky*.

Now, this is only one relatively simple example how hidden global scope can mess up your tests, there are many more possibilities, and you'd generally want to avoid them.

As a general programming advice, you'd want your modules / classes to have as little state as possible, and if state is needed, relegate it to a module / class that is specifically designed to deal with state.

## Unit tests vs. integration tests

At this point you may ask: Do unit tests completely replace integration tests? Can you fully test an application only
with unit tests?

The answer, unfortunately, is no. Unit tests test that the individual components work correctly on their own. If you 
have a bug, they tell you exactly where said bug is, and what the nature of it is.

Integration tests (or however you want to call them) test larger parts or the whole of the application, to make sure
that the components have been wired up correctly. As such, integration tests don't provide *full* coverage, but enough
to let you know if you have a wiring failure.

## Conclusion

The answer to the question posed in the title is simple: testing is hard because you have to have testable code first.
It is no use screaming at developers for not writing tests if the architecture of the application doesn't lend itself
to testing. You first have to get rid of  weeks, months, years, even decades worth of technical debt, otherwise the 
whole testing process is going to be very very painful.

Now, you don't need to do this all at once. Take your most critical application parts and start with those, then
work your way from there.

## Further reading

- [Don't Look For Things! by Misko Hevery](https://www.youtube.com/watch?v=RlfLCWKxHJ0)
- [Global State and Singletons by Misko Hevery](https://www.youtube.com/watch?v=-FRm3VPhseI)
- [Unit testing by Misko Hevery](https://www.youtube.com/watch?v=wEhu57pih5w)
- [The Clean Code series on this blog](/tags/clean-code)
