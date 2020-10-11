---
slug: make-everything-final
guid: https://pasztor.at/blog/make-everything-final/
authors:
- janos
categories:
- Technology
date: "2019-02-04T00:00:00Z"
publishDate: "2019-02-04T00:00:00Z"
summary: 'I have a pretty radical suggestion for clean code: make all your classes final!'
images:
- posts/make-everything-final.png
preview: posts/make-everything-final.jpg
tags:
- Software Development
title: Make all classes final!
---

Wait, what? Now I must have surely lost my marbles, right?

Well, let's back up a bit. What is final and why am I making such a recommendation?

In Java and other OOP the `final` keyword can be used to prevent a class from being inherited from. This is an effective
method to lock down a class. It works like this:

```java
final class MyClass {
}
```

If you now try to extend this class, you will get an error:

```java
class MySubClass extends MyClass {
}
```

So, why am I claiming you should make most of your classes final? 

## The Open-Closed principle

If you have heard about the S.O.L.I.D. principles before you may be thinking that I've completely lost it. Specifically,
my suggestion seems to violate the Open-Closed principle (OCP).

There are multiple authors who described this principle, but all of them boil down to the same point:

Software Development Open-Closed Principle
: *Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification.* &mdash; Robert C. Martin*

The reason for the open part should be pretty obvious: if a module / class / etc is not open, there is no way to extend
it with additional functionality without modifying the code of the original module.

When you want a long term maintainable code base having to touch existing modules just to implement new functionality
is evil. Because of a new feature you would have to touch old code. In other words, you implement a new feature and you
may end up breaking some old functionality. This is especially true if you do not have tests.

For example, here's a class that is not open:

```java
class TemplateRenderer {

    public String render(BlogPost blogPost) {
      //...
    }
}
```

This `TemplateRenderer` accepts only BlogPost objects, so if you wanted to render something else using the same 
template rendering mechanism, you would be out of luck.

So what about the closed part?

You see, the example above was also not closed enough. If I didn't want to touch the original code, I could do this:

```java
class ExtendedTemplateRenderer extends TemplateRenderer {
  @Override
  public String render(BlogPost blogPost) {
    if (blogPost instanceof VideoPost) {
      //Render with video
    } else {
      return super.render(blogPost);
    }
  }
}
```

The `VideoPost` would, by necessity, extend `BlogPost`, even though it might not even have a post text attached to it:

```java
class VideoPost extends BlogPost {
  @Override
  public String getText() {
    return "";
  }
  
  public String getVideoUrl() {
    //...
  }
}
```

It walks like a hack, it quacks like a hack, it is a hack. If this codebase lives long enough, there will be layers
upon layers upon layers of these hacks. If you change something *upstream* in the original `TemplateRenderer`,
it could break the whole chain.

The `TemplateRenderer` was never *designed* to be used this way, yet it is. The original author failed to properly
*close* the implementation.

## Closed by default

What does all this have to do with `final`? Why are we even talking about it? Let me explain.

Classes, by default, can be inherited from in many OOP languages. This leaves them somewhat open to abuse as seen above.
I find that most class authors don't even think about how a certain class will be inherited from. In other words, the
class is not *designed* to be inherited from.

If a class is not designed to be inherited from, inheritance, I think, should not be allowed. When a class then needs to
be extended by design, the final keyword can be removed. However, [as I argued before](/blog/oop-misunderstandings), it
is better to use composition rather than inheritance to make classes open for extension.

Let's fix the example above. First of all, let's create a common interface that all post types have to implement:

```java
interface Post {
  String getTitle();
}
```

That's it, the title is the only required part. Then we can also create a rendering strategy as follows:

```java
interface TemplateRenderingStrategy<T extends Post> {
  String render(T post);
}
```

This template rendering strategy could be unique to each post type, so this would be a good solution:

```java
class VideoPostRenderingStrategy
  implements TemplateRenderingStrategy<VideoPost> {
  
  public String render(VideoPost post) {
    //...
  }
}
```

This rendering strategy can then be passed to the renderer as a way to extend its functionality.

## Conclusion

We humans are forgetful, we sometimes don't think, and when deadlines are looming we are sometimes willing to go to any
lengths to get the job done *quickly*. Inheritance often seems like a tempting solution to introduce an ugly hack just 
to meet the deadline.

Having `final` in your classes by default prevents them to be abused like that. In fact, I find that I could often
live completely without inheritance by just using interfaces when needed.

## Sources:

- [Robert C. Martin: The Open-Closed Principle](https://web.archive.org/web/20060822033314/http://www.objectmentor.com/resources/articles/ocp.pdf)
- [Craig Larman: Protected Variation: The Importance of Being Closed](http://codecourse.sourceforge.net/materials/The-Importance-of-Being-Closed.pdf)