---
title: Mockito is Bad for Your Code
slug: mockito-is-bad-for-your-code
guid: https://pasztor.at/blog/mockito-is-bad-for-your-code/
summary: Mockito, a tool to make mocking easy, is actively encouraging test code. Here’s why.
authors:
- janos
categories:
- Technology
images:
- posts/mockito-is-bad-for-your-code/social.png
tags:
- Software Development
- Java
date: "2020-08-12T00:00:00Z"
publishDate: "2020-08-12T00:00:00Z"
---

I have recently come across a project in Java with quite a high test coverage. While having a high test coverage is admirable, I discovered something disturbing: all the tests were implemented with [Mockito](https://site.mockito.org/), a library that is supposed to make mocking dependencies easy.

*&ldquo;Wait a minute,&rdquo;* &mdash; you say &mdash; *&ldquo;isn’t mocking dependencies state of the art?&rdquo;* Yes, unfortunately, it is. Mocking is commonplace in the Java world, and Mockito is probably one of the most used libraries to this end. It’s also bad.

*&ldquo;Here we go again, bashing something popular,&rdquo;* &mdash; you say? Well, yes. We will be bashing Mockito. Not because it’s bad *per se*, but because it encourages writing bad tests. In this blog post, I want to demonstrate that overusing mocking will cause fragile tests and frustration in your team.

{{% warning %}}
**Bias warning:** I am a follower of the [classicist/Detroit school of testing](https://medium.com/@adrianbooth/test-driven-development-wars-detroit-vs-london-classicist-vs-mockist-9956c78ae95f). If you follow a different school of thought, this post may enrage you.
{{% /warning %}}

## Why do we test?

Before we dive in, let’s ask ourselves the question: *why do we test?*

![An image of Mr. T, the african-american actor playing B.A. Baracus in &ldquo;The A-Team&rdquo; with a black mohawk and a black beard, with large earrings, pointing at you. The text on the picture says: &rdquo;To prevent bugs, fool&rdquo;](posts/mockito-is-bad-for-your-code/to-prevent-bugs-fool.jpg)

*&ldquo;To prevent bugs, of course!&rdquo;* &mdash; you say. OK, let me rephrase the question: why do we *write automatic tests?* After all, if the goal was *just* to prevent bugs, we could test it once, manually, right?

You see, automated tests are there to give you a safety net for refactoring. You write them once to verify the initial correctness of your code, and then they stay there to verify the correctness of any change you make during a refactor.

*&ldquo;I’m never going to refactor this code!&rdquo;* &mdash; you may think. That’s only true if you manage to get it perfect the first time, *and* there are no subsequent tasks that require a change in the current code.

Let’s say you have a bunch of user stories. Are you going to think of *all* the requirements *all* the time? Are you 100% sure you *never* miss a single requirement? Either you have a very small project, or devote a significant amount of brainpower to juggling the requirements in your head. **Shouldn’t you focus on the user story you are working on *right now*?**

Trying to design *absolutely everything* upfront doesn’t scale. Instead, an iterative approach yields quicker and better results. However, an iterative approach also comes with refactoring as you purposely ignore future requirements.

I’m not saying that having an architecture plan upfront is a bad idea, quite the contrary! However, trying to design an application down to the last minuscule detail smells a lot like a *waterfall* approach. You know what those projects have a *lot*? Missed deadlines. Exploding costs. Frustrated employees.

To sum it up, tests are there to save your bacon both before and after the project ends. Either write them properly or don’t write them at all. If you have bad tests, they will just give you a false sense of security. 

## Frameworks should encourage good code

Now we come to the question of tooling. What tools do you use to test your code? This is a critical decision as our testing frameworks and tools will be tightly coupled with our test code by necessity.

Our tools are like parents raising children: good behavior should be reinforced, bad behavior should be discouraged. In other words, our frameworks and tools should make writing good tests easy and writing bad tests hard.

In a way, the API of our testing framework *influences* the way we write tests. We tend to favor the API’s that are easy to use. We tend to ignore APIs that are confusing, require writing a lot of boilerplate code, or are just simply hard to use.

If a testing framework makes it *easy* to write robust tests that hold up well in the face of changing internal implementations. But we’ll come back to that later.

## How Mockito works

*&ldquo;Enough with the talk already, show me the code!&rdquo;*

All right, all right. Let’s take a look at a simple example with Mockito. Let’s say we have a controller:

```java
public class UserCreateApi {
    @Autowired
    private UserStorage userStorage;

    public Response create(Request request) {
        User user = new User(
            request.email
        );
        userStorage.create(user);
        return new Response(
            user
        );
    }
}
```

OK, how do we test this the Mockito way?

```java
public class UserCreateApiTest {
    @Test
    public void testUserCreation() {
        //given
        final UserStorage userStorage =
            mock(UserStorage.class);

        final UserCreateApi api=
            new UserCreateApi(userStorage);

        //when
        final UserCreateApi.Response response =
            api.create(
                new UserCreateApi.Request(
                    "foo@example.com",
                    "asdfasdf"
                )
            );

        //then
        verify(userStorage)
            .create(
                new User("foo@example.com")
            );
    }
}
```

This looks OK at first glance, but there’s a *lot* wrong with it. First of all, what does this method *really* test? It tests if the `create` method on the user storage was called with a user object. That’s it.

Are you going to find a user story in your Scrum board that says *&ldquo;user API should call the `create` method&rdquo;*? No, of course not. Your user story will say, *&ldquo;user API should store user in the database.&rdquo;*

What’s worse, this test is *fragile*. Let’s say we add a method to `UserStorage` called `createAndReturn`:

```java
public interface UserStorage {
    void create(User user);

    User createAndReturn(User user);
}
``` 

Any *real* implementation will be *forced* to implement this method otherwise the code will not compile. With that assumption in mind, let’s change the code of the API:

```java
public class UserCreateApi {
    @Autowired
    private UserStorage userStorage;

    public Response create(Request request) {
        User user = new User(
            request.email
        );
        user = userStorage.createAndReturn(user);
        return new Response(
            user
        );
    }
}
```

The code will still fulfill the business requirements. It will store a user in the database. What happens to our test code, whoever?

```
java.lang.NullPointerException
```

Womp, womp. Mockito simply *ignores* the fact that there’s a method and simply returns `null`.

*&ldquo;But that’s not a problem,&rdquo;* &mdash; you say &mdash; *&ldquo;we can just change the tests!&rdquo;* 

Remember what we said at the beginning about the *purpose* of the tests? They are there to save our bacon when we refactor. If we have to change the live and test code *at the same time*, then why did we have tests in the first place?

Not to mention the fact that in the above example, you are left rewriting every single piece of code that uses Mockito. Remember:

> The easiest way to create *technical debt* without touching production is to *write tests*.

If your tests are hard to maintain and break all over the place, some colleagues will go: *&ldquo;testing is too hard, testing is bad, testing is useless.&rdquo;* They will be right! If every single refactor breaks the tests, then your tests are *indeed* useless! I would much rather not have *any* unit tests and rely solely on end to end tests instead of dealing with a bunch of fragile tests.

## What should we do instead?

Mockito has its uses. In very limited, fringe cases. Its API and documentation *encourage* writing fragile tests, and it should not be the centerpiece of a testing strategy.

What’s the alternative? Let’s think about it. What would we do if we didn’t have Mockito? We would be left to implement the `UserStorage` for mocking purposes ourselves. How about this?

```java
public class InMemoryUserStorage implements UserStorage {
    private final Map<String, User> usersByEmail =
        new HashMap<>();

    @Override
    public void create(
        final User user
    ) throws UserAlreadyExists {
        createAndReturn(user);
    }

    @Override
    public synchronized User createAndReturn(
        final User user
    ) throws UserAlreadyExists {
        if (usersByEmail.containsKey(user.email)) {
            throw new UserAlreadyExists();
        }
        usersByEmail.put(user.email, user);
        return user;
    }
}
```

Pretty simple, right? How does this help us? Let’s add one more method:

```java
public class InMemoryUserStorage implements UserStorage {
    private final Map<String, User> usersByEmail =
        new HashMap<>();

    //...

    public synchronized User getUserByEmail(
        String email
    ) throws UserNotFound {
        if (usersByEmail.containsKey(email)) {
            return usersByEmail.get(email);
        }
        throw new UserNotFound();
    }
}
```

Hot dang! We just implemented something we would need *anyway*! So let’s refactor our tests:

```java
public class UserCreateApiTest {
    @Test
    public void testUserCreation() throws Throwable {
        //given
        final InMemoryUserStorage userStorage =
            new InMemoryUserStorage()
        final UserCreateApi api =
            new UserCreateApi(userStorage);

        //when
        api.create(
            new UserCreateApi.Request(
                "foo@example.com",
                "asdfasdf"
            )
        );

        //then
        assertNotNull(
            userStorage.getUserByEmail(
                "foo@example.com"
            )
        );
    }
}
```

We have the exact same amount code, but this time around, we passed a complete *fake* implementation of the `UserStorage` to the API.

*&ldquo;Hey wait, but we need to write more code for the fake implementation!&rdquo;* &mdash; you say. That’s true! Let me ask you a question, though: would you rather write a couple of lines *once*, or spend your time throwing out and rewriting useless tests all over the place?

*&ldquo;OK, I get it, but who’s going to test the fake implementation?&rdquo;* &mdash; you counter. Yepp, your fake implementations *may* require tests themselves. That’s why I tend to treat my fake implementations as production code.

*&ldquo;But surely you can’t fake repositories?&rdquo;* &mdash; you may grasp at a last straw. By repositories, you, of course, mean [repositories that allow access to databases](https://docs.spring.io/spring-data/data-commons/docs/1.6.1.RELEASE/reference/html/repositories.html). Yes, you are right. Automatically generated repositories are extremely hard to fake. However, if you look at the Mockito tests for these repositories, the situation is no better: 99% of cases I have seen the tests test if a very specific SQL query is being executed. Change the query, and all your tests break.

The solution here is to either pull up an in-memory database like [HSQLDB](http://hsqldb.org/) or to leave testing the repositories to the end to end tests. Heck, you could even talk to the Docker or Kubernetes API to pull up a container with your database for you!

## Immutability is important

Before you run back to your computer and rewrite all your tests, there is one important pitfall with fakes in Java. Objects in Java are mutable. Let’s say the controller changes the value of the user after storing it:

```java
user = userStorage.createAndReturn(user);
user.setEmail("")
```

Remember, our fake implementation stores data in a hashmap in memory. If there is a `setEmail` function in user, the controller may *accidentally* change the stored version as well! That’s why [immutability](/blog/why-immutability-matters/) is an extremely important concept in avoiding such side effects.

I believe you may have something to think about. Or you could also [tell me how wrong I am on my Discord](/discord/).