---
slug: ide-matters
guid: https://pasztor.at/blog/ide-matters/
authors:
- janos
categories:
- Technology
date: "2019-03-02T00:00:00Z"
publishDate: "2019-03-02T00:00:00Z"
summary: Your IDE is your best friend when it comes to being efficient. Why do so many people not use it?
images:
- posts/ide-matters.png
preview: posts/ide-matters.jpg
tags:
- Software Development
title: Your choice of IDE matters
---

This piece started out as a rant titled *&ldquo;VIM is not an IDE&rdquo;*, but as I have talked to some people about
vim and I have to concede my point: when set up properly, VIM can be an IDE. Sort of.

But, instead of bagging on VIM, let's take a look what makes a good IDE and why it is important for clean code.

## The directory tree

Our editors and IDEs heavily influence what we optimize for. For example, if you use an IDE, a glance to the left
give you the directory tree. Therefore it is not a big hassle to create more files.

![An illustration that shows a development environment with a directory tree on the left and some source code on the right.](posts/ide-directory-tree.png "The directory tree of this website, as I'm writing it.")

You can, of course, achieve a [similar setup with vim](https://shapeshed.com/vim-netrw/), but you will need to do the config legwork to get netrw or NERDtree running. In addition, netrw, as I have tested it, doesn't work well with tabs, since tabs open on top of the split layout.

![An illustration of the editor vim with the split view option enabled.](posts/vim-split.png "A split view directory tree in vim.")

**Vim is immensely powerful, but at its core it is designed to work on one, or very few files.** Other, simpler editors have even less facilities to handle a large number of files.
 
In the last decade development of more complex software has moved to more, smaller files, self-contained, small, testable units. I have made the experience that coders using an editor that does not make handling a large number of files are usually afraid from having many files resulting in arguments like &ldquo;*In Java you need 42 classes for a Hello world!*&rdquo;.

Putting aside how nonsensical this argument is (it can be done in one class), the reason why many Java devs use so many classes is because they want to have testable units, as well as easy to replace parts. In the face of changing requirements it is better to have smaller, easy to replace units than huge chunks of code.

When I see people using &ldquo;simple&rdquo; editors that do not encourage working with multiple files well, I usually also see very, very long sections of code. Often several thousand lines long.

It's not just the editors fault, of course, but as the saying goes, if all you have is a hammer, everything looks like a nail. Again, not wanting to detract from the immense toolset vim brings to *editing individual files*.

{{% tip %}}
**Further reading:** [The Cookie Cutter Architecture](/blog/the-cookie-cutter-architecture)
{{% /tip %}}

## Code completion

Modern programming languages have a gajillion features built in. Unlike the early days we have libraries, built in or external, for almost everything under the sun and then some.

These libraries come with a large number of classes, functions and parameters. It is simply impossible to remember all but the most frequently used ones.

If your IDE analyses the code of these libraries it can come up with helpful hints what you might want to type there, speeding up the process and also eliminating long method or class names as a problem.

This, of course, cannot just simply be a dumb list of &ldquo;hey, these functions exist&rdquo; the IDE needs to *understand* the programming language you are using. Let's say you have the following piece of code:

```java
List<String> myList = new ArrayList<String>();
myList.
```

... and at this point the IDE should offer you something. The IDE here needs to understand what a `List` is, and what methods it has.

This is why so many modern IDE's have quite a bit of resource consumption. They need to build a map of your code and index it in an internal database so they have the information handy within a couple of hundred milliseconds. 

## Debugging

All reasonable programming languages and environments allow you to debug. This means that you can set a break point in your program code and make the whole system stop once you get there. While stopped, you can inspect the variables set there, and even go back in the call stack to inspect the calls that lead to the point where you are at.

![An illustration of an IDE showing the step-by-step debugging options.](posts/ide-debug.png "Debugging in a modern IDE.")

Furthermore, you can usually also go over the following code step by step, tracing the what each individual line in
the code does to the variables. This is especially important when refactoring some legacy code that you need to learn.

## Refactoring tools

![An illustration of an IDE showing the automatic renaming option.](posts/ide-refactor.png "Refactor operation in a modern IDE.")

Once your code is written, the project is done, right? Well, not so fast. Sometimes you get old code written by others, and sometimes you get old code ... well, written by you. You know the meme, when you look at the code you wrote six months ago...

Anyway, no matter if it's a typo or you just plain poorly named your class, changing it can be quite hard. That's why modern IDEs have refactoring tools. Right click, refactor, rename. However, this, again, is not simply a search and replace. If it was it would replace a lot of things it shouldn't. Instead, like before, the IDE needs to *understand* the structure of your code.

For example, let's look at this piece of Python code:

```python
foo = "bar";

def baz(foo):
    print(foo);

baz(foo);
``` 

If we refactor `foo` with a simple search and replace, it will be changed everywhere, even though our intention is most likely to only rename the variable inside the function.

This kind of refactoring often works in the most unexpected situations, such as renaming an image and having the CSS code changed to match the new image name.

However, modern IDE's go much further in helping the refactoring process. They may, for example, offer the ability to change a method signature.

![An illustration of an IDE showing how function parameters can be refactored automatically.](posts/ide-change-signature.png "The change signature option in a modern IDE.")

If we, for example, add a new parameter in this window, that change will propagate to all places where the method is used. When working in a language with static typing, or a static code analyser, it will immediately become apparent where the new parameter needs to be added. Alternatively, we can also use the &ldquo;Find usages&rdquo; option of our IDE.

## The early warning system

Another important feature modern IDEs bring is in-depth code analysis. This is especially important if you are working in a [dynamically typed language](/blog/loose-strict-static) since there is no compiler that will throw the code back in your face when you have a string but you try to use it as an int.

However, this goes further than just a simple type-check. After all, you could use a static type checker for that. The idea is that the IDE will tell you about potential mistakes *as you write them*. A good IDE will root out your bad practices. The turnaround time between making a mistake and fixing it is much faster.

![An illustration of an IDE showing warnings about code quality as the developer types.](posts/ide-static-code-analysis.png "A code analysis warning in a modern IDE.")

This, of course, also influences how you write code. If you pay attention to these kind of warnings you will stop being callous about writing potentially unsafe code.

Look, I get it when someone writes a genious piece of code that compresses several hundred lines of code down to five, but you have to realize that in most cases those five lines will be utterly unreadable. When the next guy comes along to edit something they will have a hard time to work with it.

In most cases good code is simple, boring and easy to read. That means I shouldn't have to figure out by what dark magic your variable exists. I should be able to Ctrl+Click in my modern IDE and it should get me to where that variable is defined.

And in the odd chance that you really, really need to do micro-optimization in the code instead of macro optimization, it should be really really well documented.

![An illustration of an IDE where a code quality warning is suppressed using a comment.](posts/ide-suppressed-inspection.png "A suppressed inspection in a modern IDE")

These kind of inspections are basically like a quick code review, but automated. You should use them.

{{% tip %}}
**Further reading:** [The loose, the strict and the static typing](/blog/loose-strict-static)
{{% /tip %}}

## Discoverability

This leads us right into the next topic: discoverability. When diving into some unknown piece of code readability and discoverability are the two most important aspects. Having a
[good architecture](/blog/structure-based-on-intent) is, of course, paramount, but diving into the code is still a challenge, especially when the code is a bit older and has overgone a few iterations.  

Most of us developers, at some point in our lives, get a piece of code we haven't written and the original maintainer is long gone. The first reaction, of course, is *&ldquo;This is shit. We need to rewrite it.&rdquo;* However, if that particular piece of code is making a lot of money simply rewriting it is simply not an option. Not to mention that the business rules that govern that piece of code may not even be written down and we would have to spend months trying to learn it.

This is when having a powerful IDE comes in handy. Most modern IDEs index the code and let us search very quickly. We can often even specify context, such as search for this only in comments, or only in code. We can then also Ctrl+Click our way through the codebase. 

You may be thinking that you will not deal with code someone else wrote. Guess what! If you leave your own code alone for six months down the line your own code is going to look alien to you. You will have to re-learn the structures of your project, what is where, etc.

## Automatic deployment

One issue that I come across quite frequently is the sufficiently configured and powerful development environment. Let me explain.

I recently had a developer complain that their company issued notebook was not powerful enough to run their machine learning code. *&ldquo;Why don't you use a cloud virtual server?&rdquo; &mdash; I asked. And the answer was surprising. They didn't know how to use their IDE and get their code up on the server in an efficient manner.

So, what are the options? Either you manually copy the code to the server (ugh) or you commit every change to your versioning system (git) and pull on the server (double ugh). No, none of these is a workable solution.

![An illustration of an IDE configuration window with options to automatically upload files using SFTP.](posts/ide-deployment.png "Deployment configuration in a modern IDE")

Thankfully, modern IDE's have automated deployment tools. Edit the code, the IDE automatically syncs the code to the server and by the time you get over to your server to test it, it's already there.

This also fixes another problem that appeared frequently. When people run Docker containers, they often get their code into the containers using [Docker volumes](https://docs.docker.com/storage/volumes/). Volumes allow the developer to share a folder between their host machine and the Docker container.

However, on Windows or MacOS volume mounts are frequently causing problems. This, again, is solved by the IDE by simply syncing the code into the Docker container using SSH.

## Not the tools you are looking for

These are just a few examples how IDEs help getting a better coder. You could go on about a million other things, but the point is, once learned, they become immensely powerful tools in your arsenal. They certainly helped me learn new programming languages by pointing out inefficient code or bad practices, and the time they saved me offset the cost by a factor of a hundred. 

In general, as you go into larger, team projects, having an IDE that helps you reign in the chaos is definitely a must. In my eyes &ldquo;simple&rdquo; editors are just not suitable for projects above a few thousand lines of code as they will not give you the tooling support.

And they don't even have to be expensive! [Visual Studio Code](https://code.visualstudio.com/), or
[PyCharm Community Edition](https://www.jetbrains.com/pycharm/) are just a few examples for free, powerful IDEs. Take your pick!

## You can still use vim

If you're a vim addict, you don't have to give up any of that despite using a modern IDE. Most development environments offer vim emulation, so you can use all your vim text editing goodies. There is no reason not to embrace the IDE as a powerful tool to help you with your code, but still keep your keyboard-only text editing superpowers.

## Conclusion

It is also worth mentioning that not all IDEs are created equal. Some are slow, or do not provide a benefit compared to a plain text editor. In the end you will have to make your own tool choices.

However, I find that a lot of coders, especially less experienced, voice their disdain for IDE's because they are *&ldquo;bloated and just slow me down&rdquo;*. Yes, they do require a lot of resources, and if you don't learn to use them, they indeed just slow you down. They are complex tools that need learning. Just as your favorite programming language needed learning.
