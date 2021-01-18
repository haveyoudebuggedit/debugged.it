---
slug: why-interfaces
guid: https://pasztor.at/blog/why-interfaces/
authors:
- janos
categories:
- Technology
date: "2018-09-03T00:00:00Z"
publishDate: "2018-09-03T00:00:00Z"
summary: 'I''ve received the same question from a fair number of people: why would you even use interfaces?'
images:
- posts/why-interfaces.png
preview: posts/why-interfaces.jpg
tags:
- Software Development
title: Why should you use interfaces?
---

When it comes to interfaces people tend to think that their only use is when you have multiple implementations you can
switch them out easily. Hoever, most people don't have multiple implementations of a certain functionality in their 
application. So why then would you use interfaces at all? After all the refactoring tools in our IDEs are powerful such
that we can introduce them later on...

{{% tip %}}
**Hint:** Need a quick refresher on interfaces? [Read up on the topic here!](/blog/the-curious-case-of-interfaces)
{{% /tip %}}

## Contracts, not interfaces

Most people imagine an interface to be something like this:

```java
interface ContentAuthorizer {
    boolean authorize(String userId, String contentId);
}
```

This interface describes a *method signature* that must be implemented, but there is no indication as to how this method
should behave, or (depending on your language) if null values are accepted and what exceptions are thrown. So it does
very little to actually document the expected behavior.

As many people, especially the people who question the usefulness of interfaces, realize, this isn't especially helpful.
If we don't have multiple `ContentAuthorizer` implementations, this isn't useful.

Instead, I'd like to advocate for a philosophy change. Instead of thinking about interfaces as *signature enforcement*,
think about them as *contracts*. They should describe how the implementing party has to behave and what the using party
as to pay attention to (for example, which exceptions need catching).

So a better way to write the interface above would be:

```java
interface ContentAuthorizer {
    /**
     * Decide if a certain user can access a certain piece of content and
     * return true if the user is allowed to access the content.
     *
     * @param userId    the ID of the user requesting access. Must not be null
     *                  and must contain a valid user ID
     * @param contentId the ID of the content that access is requested to.
     *                  Must not be null and must contain a valid content ID.
     *
     * @return true if the user is allowed to access, false otherwise.
     *
     * @throws InvalidUserId    if the userId parameter is null or of an
     *                          invalid format.
     * @throws NoSuchUser       if the user specified with the ID is not
     *                          found.
     * @throws InvalidContentId if the contentId parameter is null or of
     *                          an invalid format.
     * @throws NoSuchContent    if the content specified with the ID is
     *                          not found.    
     */
    boolean authorize(String userId, String contentId);
}
```

Wow, that's a lot of text for a little function like this! However, if you look at it, we defined *behavior* rather than
a signature. Before even implementing it, we thought through all the failure cases and defined proper error handling.

If you didn't do this, what are the chances you'd be too lazy to do proper exceptions and just handle everything with
a `NullPointerException` or an `InvalidParameterException`? What are the chances that, in order to find out what 
exceptions the underlying code is throwing, you'd actually have *read through that code*?

**The purpose of contracts is to define an internal API that you can use without thinking about the underlying 
implementation. Like a well-written legal document, it lays out exactly how each party is supposed to behave.**

## Testing

Now, let's go a step further. Let's assume you not only want a good structure but you also want to test your
application. As [discussed previously](/blog/why-testing-is-hard) one of the probably easiest tests to write is a
unit test.

A unit test is called that because it tests a unit (or class in our case) *in isolation*. What does this mean? Let's 
assume we want to test a controller such as this:

```java
class BlogPostController {
    public ViewModel getLatestBlogPosts() {
        //...
    }
}
```

This controller obviously has some dependencies which we, of course, [inject these dependencies](/blog/clean-code-dependencies):

```java
class BlogPostController {
    private BlogPostFetchBusinessLogic blogPostFetchbusinessLogic;
    //...

    public BlogPostController(
        BlogPostFetchBusinessLogic blogPostFetchbusinessLogic
        //...
    ) {
        this.blogPostFetchbusinessLogic = blogPostFetchbusinessLogic;
        //...
    }
    
    //...
}
```

We have two cases now: either `BlogPostFetchBusinessLogic` is an interface or it is an actual implementation. Let's look
at how our tests would look like in both cases. First with the interface:

```java
class BlogPostControllerTest {
    private BlogPostController createController() {
        return new BlogPostController(
            new FakeBlogPostFetchBusinessLogic()
        );
    }
    
    class FakeBlogPostFetchBusinessLogic implements BlogPostFetchBusinessLogic {
        //...
    }
}
```

So we are passing an actual, simplified implementation of the fetch business logic. This fake business logic has no
other dependencies so it's rather easy to instantiate for the purposes of a test.

Now, how does the same code look when we instantiate the actual implementation?

```java
class BlogPostControllerTest {
    private BlogPostController createController() {
        return new BlogPostController(
            new BlogPostFetchBusinessLogic(
                new BlogPostStorage(
                    new DatabaseConnectionFactory(
                        //database parameters
                    )
                )
            )
        );
    }
}
```

In essence, you are pulling in and testing the whole application, not just the controller. If any of the underlying
layers has a problem, that will pop up as a failure in the unit test as well. Remember, the purpose of a unit test is to
**precisely indicate where the problem is.** If 30 tests fail because you made one mistake somewhere in your storage
layer or your database isn't available, that's not going to be terribly helpful in tracking down the problem.

To conclude, if you write unit tests you **do** have multiple implementations for the same interface.

## Proxy patterns

When you embrace interfaces as a tool in your arsenal, you can do pretty neat tricks with it too. Let's say you have
a class that fetches some data from a remote API. Or, more precisely, let's use an interface:

```java
interface MyRemoteDataFetcher {
    /**
     * Fetch the remote data set by ID. As the remote data set is immutable, the method MAY return a cached version.
     *
     * ...
     */
    MyRemoteDataSet fetchRemoteData(String dataId);
}
```

As you can see, the interface is well described and could actually cache the data locally so it doesn't need to be
re-fetched every time since it won't be modified anyway.

If we now decide to put the fetching and caching logic all in one class that would be a gross violation of the 
[Single Responsibility Principle](/blog/clean-code-responsibilities). So, we can use interfaces such as this:

```java
class MyRemoteDataFetcherImpl implements MyRemoteDataFetcher {
    public MyRemoteDataSet fetchRemoteData(String dataId) {
        //fatch
    }
}

class MyCachingProxyRemoteDataFetcherImpl implements MyRemoteDataFetcher {
    private MyRemoteDataFetcher actualFetcher;
    
    public MyCachingRemoteDataFetcherImpl(
        MyRemoteDataFetcher actualFetcher
    ) {
        this.actualFetcher = actualFetcher;
    }
    
    public MyRemoteDataSet fetchRemoteData(String dataId) {
        //Use the actual fetcher to fetch if the data is not cached
    }
}
```

{{% tip %}}
**Hint:** Usually you want to call your implementation something more descriptive, such as embedding the library the implementation is using. One good example would be `UnirestRemoteDataFetcher` and `InMemoryCachingRemoteDataFetcher`.
{{% /tip %}}

As you can see, one implementation of the interface uses the other implementation. We can then configure our dependency
injector to chain them together to let the application cache the data. This way we won't violate the SRP and we also
won't have to touch our fetcher implementation if we later on decide to put in the caching logic.

{{% warning %}}
**Warning!** In the spirit of *contracts* caching should only be added if the contract allows for it! If you add caching without the upper layer expecting it, you may break the application!
{{% /warning %}}

## Conclusion

I could probably outline a couple more use cases around interfaces, but I'm hoping that you'll embrace it as a very
useful tool in your arsenal. Thinking ahead and defining your internal APIs can save you a ton of time later on.