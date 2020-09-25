---
slug: unit-testing
guid: https://pasztor.at/blog/unit-testing/
authors:
- janos
categories:
- Technology
date: "2019-03-19T00:00:00Z"
publishDate: "2019-03-19T00:00:00Z"
summary: Having anxiety attacks before deployment? Do you keep breaking stuff that already worked? Unit tests are here to help!
images:
- posts/unit-testing.png
preview: posts/unit-testing.jpg
tags:
- Software Development
title: A beginners guide to unit testing
---

When you start to work on a software project, everything seems easy. Launch some scaffolding script, write a bit of 
code, make a sexy UI and there you go, the first feature is ready in no time. Your boss is happy, you are happy, 
it's a green field project.

However, over time things seem to slow down. As the application grows it becomes trickier to to test everything
before deployment and fixing one thing results in breaking two other things. New requirements come in, you need to 
change your application, and everything just becomes a mess. A couple of months down the line you utter the famous
sentence: *&ldquo;We need to rewrite this.&rdquo;*

The reason for this slowdown is that as the code grows and gets changed to match the updated requirements, there is 
nothing to ensure that what worked last week still works after the change. You could just say *&ldquo;don't change
the damn requirements!&rdquo;* but honestly, that's not realistic. The requirements *always* change.

Let's think about it: how did you ensure code quality? You *tested it*. By hand. And it's terribly inefficient. We as
developers, who automate things, absolutely *hate* monotonous, repetitive work. So we forget to test. Or, even worse,
we are under pressure to deliver, so we skip testing.

However, instead of skipping tests we should automate them. We should have a suite of tests that automatically run
for every deployment, or even better, as we code, to make sure we didn't break things. After all, the sooner we 
discover the mistake the easier it is to fix.

## How do you test

Now, if you think of testing lots of people go along the lines of just launching the application, clicking around in it
and checking if it still works. This can be done with tools like [Selenium](https://www.seleniumhq.org/). These kind
of tests test the application end-to-end. It's like taking a car and driving it. If it works, you know it works.
However, if it doesn't work you have to take it to a mechanic and have it taken a part.

The mechanic takes it apart and tests the individual parts in isolation. For example, they might take the radio out and
put it on a test bench. This is akin to what we call *unit tests*.

Unit tests test a *unit* in isolation, disconnecting any and all external dependencies. While an end-to-end or 
integration test tells you that something is wrong, they don't tell you what exactly is wrong. Unit tests tell you
exactly what is wrong, but they don't tell you if the application is put together correctly.

It is important to note that the purpose of unit tests is to tell you with absolute certainty that the component
it tests is broken. To ensure that goal is met it is very important that all external dependencies, such as databases,
external service providers, but even internal dependencies be disconnected, otherwise factors like network problems
could falsify the tests. A test that requires a running database server is not a unit test.

## Writing your first test 

But enough of the words, let's write our first test. Let's take the rather simple problem of the Fibonacci sequence.
The code looks like this, here in Python:

```python
def fibonacci(n: int) -> int:
    if n > 2:
        return fibonacci(n-1) + fibonacci(n-2)
    return 1
```

Pretty simple, right? So let's test it. In Python there is a built-in tool called
[`unittest`](https://docs.python.org/3.7/library/unittest.html). If we follow the example in the documentation our
tests could look like this:

```python
import unittest

class TestFibonacci(unittest.TestCase):
    def test_numbers(self):
        self.assertEqual(1, fibonacci(1))
        self.assertEqual(1, fibonacci(2))
        self.assertEqual(2, fibonacci(3))
        self.assertEqual(3, fibonacci(4))
        self.assertEqual(5, fibonacci(5))
        self.assertEqual(8, fibonacci(6))
        self.assertEqual(13, fibonacci(7))
        self.assertEqual(21, fibonacci(8))
        self.assertEqual(34, fibonacci(9))
        self.assertEqual(55, fibonacci(10))


if __name__ == '__main__':
    unittest.main()
```

If we now run this Python file we see the following:

```
.
----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
```

Pretty simple, right? We write a function, write a test for it, run the test for every deployment and that's it, we have
tested our application.

## Disconnecting dependencies

This is usually where most tutorials on unit testing end. Coincidentally, this is also where unit testing becomes 
hard. Remember, in the beginning I mentioned that unit tests test a single unit, without external dependencies.

So, how do we do that? Well *it depends*. It mainly depends on your programming paradigm. Let's look at OOP first.

### Object Oriented Programming

Let's say we have a `DataProcessor` class that takes input from a queue and sends the output to a backend. Its task
is to provision and deprovision services.

In essence, we can describe the `DataProcessor` like this:

```python
class DataProcessor:
    queue: Queue
    backend: Backend

    def __init__(self, queue: Queue, backend: Backend):
        self.queue = queue
        self.backend = backend

    def process_one(self):
        task = self.queue.fetch()
        if task is None:
            return
        try:
            if task.type == TaskType.PROVISION:
                self.backend.provision(task.customer_id)
            else:
                self.backend.deprovision(task.customer_id)
            self.queue.complete(task)
        except BackendException:
            self.queue.fail(task)
        except Exception as e:
            self.queue.fail(task)
            raise e
```

Observe the constructor (`__init__`). This constructor takes two parameters and stores them in member variables.
The dependencies are *injected* from the outside. This is called **dependency injection**. In order to use this class
we now need to pass it one `Queue` and one `Backend`.

The trick here is that both `Queue` and `Backend` should be abstract classes or interfaces, depending on what your
language of choice supports. In Python the `Queue` could look like this:

```python
class Queue(ABC):
    @abstractmethod
    def fetch(self) -> Optional[Task]:
        pass

    @abstractmethod
    def complete(self, task: Task) -> None:
        pass

    @abstractmethod
    def fail(self, task: Task) -> None:
        pass
```

We could define `Backend` in a similar fashion. Now, in order to use the `DataProcessor` we need to write an 
*implementation* for this abstract class. This is going to the real queue and the real backend, such as `RabbitMQQueue`
and `RESTBackend` or something like that.

However, in order to *test* the `DataProcessor` we will need a *second* implementation for testing purposes. Let's
call these `InMemoryQueue` and `InMemoryBackend`. The queue could look like this:

```python
class InMemoryQueue(Queue):
    new_tasks: List[Task] = []
    completed_tasks: List[Task] = []
    failed_tasks: List[Task] = []

    def fetch(self) -> Optional[Task]:
        if len(self.new_tasks) > 0:
            item = self.new_tasks[0]
            self.new_tasks = self.new_tasks[:-1]
            return item
        return None

    def fail(self, task: Task) -> None:
        self.failed_tasks.append(task)

    def complete(self, task: Task) -> None:
        self.completed_tasks.append(task)
```

This queue stores all its data in memory. We can now add a couple more functions to manage the class variables, but the
point is that we now have a fully functional second implementation for the queue that we can use for testing purposes.
We can do the same to the `Backend` and then get right to testing.

We write our first test where we test if a task can be provisioned successfully:

```python
class TestDataProcessor(unittest.TestCase):
    def test_success(self):
        # Setup
        queue = InMemoryQueue()
        backend = InMemoryBackend()
        processor = DataProcessor(queue, backend)
        task = Task(TaskType.PROVISION, 3)
        queue.enqueue(task)

        # Execute
        processor.process_one()

        # Assert
        self.assertEqual([3], backend.get_customer_ids())
```

As you can see we divided the test into three phases: setup, execute and assert. In the setup phase we constructed
the object chain. The execute is responsible for executing the test, and the assert checks if the assumption we made
holds true with our code.

### Functional programming

What about FP you ask? Well, the situation is almost the same. If you don't consider testing your processor function
might look something like this:

```python
def process_one():
    task = fetch_task()
    if task is None:
        return
    success: bool
    if task.type == TaskType.PROVISION:
        success = provision_user(task.customer_id)
    else:
        success = deprovision_user(task.customer_id)
    if success:
        complete_task(task)
    else:
        fail_task(task)
```

However, as you may notice, the `fetch_task` function call itself is implicit, so there is no way for us to mask the 
dependency. So we will create a closure that will provide the dependencies:

```python
def create_process_one(
        fetch_task: Callable[[], Optional[Task]],
        complete_task: Callable[[Task], None],
        fail_task: Callable[[Task], None],
        provision_user: Callable[[int], None],
        deprovision_user: Callable[[int], None]
):
    def process_one():
        task = fetch_task()
        if task is None:
            return
        success: bool
        if task.type == TaskType.PROVISION:
            success = provision_user(task.customer_id)
        else:
            success = deprovision_user(task.customer_id)
        if success:
            complete_task(task)
        else:
            fail_task(task)

    return process_one
```

If we now want to use this function, we can do it like so:

```python
process_one = create_process_one(
    fetch_task,
    complete_task,
    fail_task,
    provision_user,
    deprovision_user
)

process_one()
```

We are doing the same as we did before with OOP, separating out the place of passing the dependencies from the place
where the function is actually called. The parameters (`fetch_task`, `complete_task`, etc) will, of course, be 
functions that deal with the queue.

In this case our test will be a bit longer as we will need to deal with state handling, but it's not too terrible:

```python
class TestDataProcessor(unittest.TestCase):
    def test_success(self):
        # Setup
        new_tasks: List[Task] = []
        completed_tasks: List[Task] = []
        failed_tasks: List[Task] = []
        customer_ids: List[int] = []

        def fetch_task() -> Optional[Task]:
            if len(new_tasks) > 0:
                item = new_tasks[0]
                new_tasks.remove(item)
                return item
            return None

        def complete_task(task: Task) -> None:
            completed_tasks.append(task)

        def fail_task(task: Task) -> None:
            failed_tasks.append(task)

        def provision_user(customer_id: int) -> None:
            if customer_id not in customer_ids:
                customer_ids.append(customer_id)

        def deprovision_user(customer_id: int) -> None:
            if customer_id in customer_ids:
                customer_ids.remove(customer_id)

        process_one = create_process_one(
            fetch_task,
            complete_task,
            fail_task,
            provision_user,
            deprovision_user
        )

        task = Task(TaskType.PROVISION, 3)
        new_tasks.append(task)

        # Execute
        process_one()

        # Assert
        self.assertEqual([3], customer_ids)
```

Truth be told, this isn't the best example for a functional-style approach, but you get the idea. (Did I mention that
you should pick the programming paradigm that best suits the task, and not based on some ideology?)

Anyway, you can see that the dependencies can easily be separated in FP as well, which makes unit testing a breeze.

## Fakes, mocks, and relying on internal implementation details

It is important to note though that the dependency implementations we created to help our testing efforts, such as the
`InMemoryQueue` or the `InMemoryBackend` should behave the same way as the actual production classes will. This is
called a *fake*: it resembles the production setup as close as possible.

If we take our `InMemoryQueue` example, it actually contains three queues: one for the new tasks, one for the completed
ones and one for the failed ones. If your actual queue does not behave the same way, you might end up with issues down
the line.

This could be for various reasons, for example, the queue could *retry* a task and your code may not be prepared to 
handle it.

A different, and arguably worse approach would be to create a *mock*: a class that is specifically created to receive
the calls as they are in the actual implementation and doesn't bother to provide a full implementation.
 
This falls under the general category of relying on the internal implementation details of the production code.
To illustrate this problem let's take a look at a particularly bad example:

```python
class Foo:
    def __init__(self, bar):
        self.bar = bar
        
    def do_something(some_switch: bool):
        if some_switch:
            return bar.baz()
        return "Hello world!"
```

As you can see, if we call `do_something` with `False` as a parameter, bar will not be invoked. But we only know that
if we look at the code. So a bad test might do something like this:

```python
def test_with_false_switch(self):
    # Setup
    foo = Foo(None)
    # Execute
    result = foo.do_something(false)
    # Assert
    self.assertEqual("Hello world!", result)
``` 

This test doesn't bother passing the dependency to `Foo`. However, if the implementation of `do_something` is changed
to involve `bar` for some reason, all the tests that do this break. The tests are fragile.

Tests should be written based on the public interface of the production code *only* and should never rely on the
internal workings of said production code.

Similarly, passing incomplete implementations of dependencies (mocks, etc) is a terrible idea if you want to have a
maintainable codebase.

## The test-complexity issue

You may also notice that our FP example got a little out of hand and would need some refactoring. This highlights a very
important point: the easiest way to create technical debt without touching the production code is via tests. You *have*
to treat your tests with the same measure of quality as your production code. That includes refactoring it, creating
abstractions, etc. If you just copy-paste your instance creations to 50 different places you are going to absolutely
hate yourself down the line. Use factories, use the tools you would normally use in your production code to ensure
things remain nice and tidy.

## Introducing TDD

Once you play around with testing a bit more one question might pop up in your mind: how do you ensure that all 
your code paths are covered? Did you just anticipate the happy case, or did you do error handling as well?

Being an intelligent person, you can probably think through all use cases, but I wouldn't be so sure about Dave
over there. Also, when the deadline is looming you don't have enough time to deal with it. So let's find an easier
way.

For a moment let's go back to the textbook examples and assume we have the task of writing a function that splits
a positive number into its prime components. So if we call `prime_components(66)` we should get `[2, 3, 11]`.

Furthermore, we will follow these three rules formulated by [Uncle Bob](http://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd)

1. We are not allowed to write any production code unless it is to make a failing unit test pass.
2. We are not allowed to write any more of a unit test than is sufficient to fail; and compilation failures are failures.
3. We are not allowed to write any more production code than is sufficient to pass the one failing unit test.

I will add one more asterisk: we will do refactoring in the process.

So let's start. We will start with a test:

```python
class TestPrimeComponents(unittest.TestCase):
    def test_one(self):
        self.assertEqual([1], prime_components(1))
```

Very simple, right? Let's run it:

```
Error
Traceback (most recent call last):
  File "/usr/lib/python3.6/unittest/case.py", line 59, in testPartExecutor
    yield
  File "/usr/lib/python3.6/unittest/case.py", line 605, in run
    testMethod()
  File "/home/janoszen/opsbears/unit-testing/primes.py", line 6, in test_one
    self.assertEqual([1], prime_components(1));
NameError: name 'prime_components' is not defined
```

OK, cool, we don't have a `prime_components` function, no surprise there. The reason why we need to *run* the test and 
see it fail is to make sure that the test is actually, truly failing and we didn't make a mistake. So let's define
the `prime_components` function:

```python
def prime_components(n: int) -> List[int]:
    return [1]
```

Wait, what? Yes, you are reading this correctly. We are returning a static value. After all, we are only allowed to 
write enough code to pass the test. Run it, all cool. Next test:

```python
def test_two(self):
    self.assertEqual([2], prime_components(2))
```

Run it, see it fail. Let's fix it:

```python
def prime_components(n: int) -> List[int]:
    return [n]
```

Now, let's continue on and venture until `test_four` where our next failure will happen:

```python
def test_four(self):
    self.assertEqual([2,2], prime_components(4))
```

Fixing it is again a braindead task:

```python
def prime_components(n: int) -> List[int]:
    if n == 4:
        return [2, 2]
    return [n]
```

... you get the picture. At the end you will arrive at a test suite that covers everything you have written code for.
Inbetween the steps **you refactor** to make your code nicer and more efficient. The tests ensure that you will not
break anything that worked so far. Essentially, you go through three steps:

1. Write a test and see it fail
2. Write the code to fix the test
3. Refactor the code

This is called *T*est *D*riven *D*evelopment. In my eyes it doesn't always make sense, but when you are stuck at a 
particularly complex piece of code it's a nice tool in your arsenal to use when needed, so I would recommend practicing
it.

## When to use unit tests

Since I'm at the topic of making sense, to unit tests always make sense? Sure enough, they are the best possible tool to
stabilize your code and make sure you can, with reasonable certainty, assume your code works on a unit level.

You will, of course, need higher level tests (integration) that ensure that your units are configured
together correctly. After all what's the point of having bug-free units if you then wire them together incorrectly?

In some cases it makes sense not to have unit tests at all. For example, when I'm doing exploratory coding, trying to
figure out how my modules will fit together, I often throw out whole units, or even change their API completely. If I 
had to then change my unit tests all the time, while I'm not even sure how they are going to work. In this case
I use higher level tests to hold the whole thing together and go back later to fill in the blanks. At times I even 
throw out and rewrite the contents of the units to make sure I cover everything with tests.

## What is a unit?

Now, just what is a unit? If you program in OOP, you might think that a class is a unit, and that can certainly be true
in some cases. But what about the factories you need to create an instance of that class? Or the data classes that
ship the data from and to that class?

These can all be part of your unit. It doesn't have to be a single class, but it has to be small and self contained.
If you try and lock down everything to a single class you will always have a moral dillema when you need to intrude on
that unit.

## Summary

To be quite frank with you, there are many views out there on the merits of unit testing, integration testing and TDD.
One of the most prominent opinions arguing against TDD is [David Heinemeier Hanssons](https://dhh.dk/2014/tdd-is-dead-long-live-testing.html).
One thing is sure: unless you want to have hordes of testers sitting there and clicking through every aspect of your
application by hand using a 600 page testing manual, or ship untested code to production, you need automated testing.

I find automated testing, and unit testing in particular to be an invaluable tool in my arsenal. They give me peace of
mind that my code is in an OK state when I need to refactor something. Without tests I would spend much, much more time
on testing it. To me it is definitely worth the up-front investment, especially considering that filling in tests
afterwards is something that is often talked about but never happens.

It does not matter *how* you test your code as long as you do *have* tests. Different industries have different
testing cultures and methodologies. Does it make sense to unit test a game? Probably not. You would want to have
higher level tests for that. How about a financial routine that calculates things? Hell yeah.

If you want to further dive into testing, I would recommend the following materials:

- [Clean Code: Dependencies](/blog/clean-code-dependencies.html)
- [What the *** is an IoC container?](/blog/what-is-the-ioc-container.html)
- [Why testing is hard?](/blog/why-testing-is-hard)
- [Misko Hevery &ndash; Mr Testable vs. Mr Untestable](https://www.youtube.com/watch?v=wEhu57pih5w)
- [Misko Hevery &ndash; Don't look for things!](https://www.youtube.com/watch?v=RlfLCWKxHJ0)
- [Eric Elliot &ndash; Mocking is Code Smell](https://medium.com/javascript-scene/mocking-is-a-code-smell-944a70c90a6a)
- [Martin Fowler &ndash; Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)