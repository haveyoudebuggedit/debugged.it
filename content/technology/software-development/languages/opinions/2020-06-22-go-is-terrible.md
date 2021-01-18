---
categories:
- Technology
slug: go-is-terrible
guid: https://pasztor.at/blog/go-is-terrible/
date: "2020-06-22T00:00:00Z"
publishDate: "2020-06-22T00:00:00Z"
summary: Go certainly caught a lot of attention. Let's look at the bad parts!
images:
- posts/go-is-terrible.png
preview: posts/go-is-terrible.jpg
tags:
- Software Development
- Golang
authors:
- janos
title: Go is a terrible language
---

As a developer it's hard to ignore [Go](https://golang.org/) nowadays. Powering software like
[Docker](https://www.docker.com/) and [Kubernetes](https://kubernetes.io/) it has risen to rapid prominence. However,
if one looks at the [popularity graphs](http://pypl.github.io/PYPL.html) of
[the language](https://www.tiobe.com/tiobe-index/) Go is very far from the most popular, or even the fastest growing
programming language.

{{% tip %}}
**Do you want a different opinion?** Read my post about [why Go is awesome](/blog/go-is-awesome)!
{{% /tip %}}

Why is it then that everybody seems to be talking about and hiring for Go? Seemingly everybody wants to use Go, from
system-level engineering to building webshops? Is this just a hype curve and is Go even suitable for the tasks it is 
being used for?

I have recently written an [SSH server that launches containers](https://github.com/janoszen/containerssh) in Go.
The project has certainly grown to quite a large size, and I have also sent a
[pull request to Golang itself](https://go-review.googlesource.com/c/crypto/+/236517) to fix a bug I found after having
gathered substantially more experience than a `Hello world!`.

In this article I'm going to take a look at the **bad parts**: the language design flaws, the parts where Go needs to
mature more, down to the plain annoying stupid things.

However, it's not all bad: Go has some [awesome features](/blog/go-is-awesome) that make it an invaluable tool for many
applications. [&ldquo;Go&rdquo; read my other article](/blog/go-is-awesome) if you are interested in those. (Pun totally
intended.)

Still here? Good. Let's look at the nasty bits. Keep in mind that this post was written for Go 1.14. Things might
have changed in the meantime.

## Error handling

Go has no [exceptions](https://en.wikipedia.org/wiki/Exception_handling). While there are very valid criticisms against
exceptions in my opinion the method Go has chosen to deal with errors is way worse.

You see, in Go you can declare a function like this:

```go
func doSomething() error {
    return errors.New("this is an error")
}
```

When you call this function you have the *option* to handle this error:

```go
func doSomethingElse() error {
    err := doSomething()
    if err != nil {
        return err
    }
    //More things to do
}
```

Oh boy, there is so much to unpack here. First of all, nothing forces you to handle this potential error. Sure, IDEs
like [Goland](https://www.jetbrains.com/go/) will warn you about it, but the compiler doesn't force you to handle this
error. (This problem can also be partially mitigated by linters that check your source code for accidental omissions
like this.)

Going further, the error is basically a string in 99.9% of cases. Yes, `error` is an interface that you can implement
any way you want, but most Go code I have seen doesn't utilize this to provide typed errors. The standard way of
providing typed errors seems to be the following:

```go
var MyError = errors.New("this is an error")

func doSomething() error {
    return MyError
}

func doSomethingElse() error {
    err := doSomething()
    if err != nil {
        if err == MyError {
            // Handle MyError specifically
        } else {
            // Handle other errors
        }
    }
    //More things to do
}
```

That's a LOT of boilerplate code for a simple error handling. However, the larger problem is that functions don't
declare what kinds of errors they return. When I'm using a function of a third party library that does this I'm left 
with two options:

1. I use the IDE to dive into the third party library to reverse engineer what kinds of errors it throws and then
   handle them. I have written production-grade code in roughly a dozen programming languages during my career,
   but I *never* had to do so much reverse engineering as with Go.
2. Treat the error as a string of unknown content.
 
Almost every piece of Go code I have looked into uses option 2. No wonder, it's the simpler option to code. This leads
to one of the following two patterns:

```go
if err != nil {
    return err
}
```

```go
if err != nil {
    log.Fatalf("an error happened (%v)", err)
}
```

The first option is basically equivalent to exceptions. Except that it leads to a *ton* of boilerplate code. The second
option crashes the program with an unhelpful error message.

I get the idea of forcing developers to handle errors explicitly, but most of the Go code I have seen just uses one of
these two mindless patterns.

Why can we not just simply have exceptions that catch *typed* errors? It would be so much simpler and lead to so much
less boilerplate code. Alas, it's probably too late for that now. 

## Nullability

The `nil` value in Go is usable for any [pointer type](https://tour.golang.org/moretypes/1). Pointers hold an address of
a piece of memory. In Go you can't directly allocate memory, but other programming languages like C allow you to do
that. The pointer address of `0` is a synonym for *not pointing anywhere*. Of course a memory address of `0` doesn't
[necessarily mean an invalid memory location](https://en.wikipedia.org/wiki/Zero_page) but modern compilers understand
this and translate a null pointer to the corresponding no-value type of the CPU architecture you are compiling for.

Now, in Go `nil` values are actually a problem because there is no in-language way to indicate if a value can or 
cannot be null.

```go
something := getSomething()
something.process()
```

This code may lead to a crash if `something` is `nil`. Yes, a full-on crash. You can, of course, rewrite the code
to include error handling, but it would be better if Go had learnt from the mistakes of other languages. 

## Scoping and code structure

By far one of the biggest bug bears I have with Go is the scoping. Go does not have qualifiers like `public`, `private`
or `protected`. The compiler takes all `.go` files in the same package and merges them. If a variable name, interface,
etc. is written with a lower case starting letter it is considered &ldquo;private&rdquo; and is only visible within
the package. If it is written with a capital first letter it is &ldquo;public&rdquo;.

In other words lower case things are only accessible in the same package, upper case things are globally visible.
Unfortunately there is **no way to restrict visibility within the same package**.

{{% tip %}}
**Note:** When using Go modules (which is the preferred way) one package means one directory. Other build systems like [Bazel](https://bazel.build/) allow for multiple packages per directory. This somewhat mitigates the lack of
scoping.
{{% /tip %}}

Imagine you have a data structure, and a set of functions that implement a very specific business logic. Someone who is
not familiar with the business logic might not think much of it and implement a function in the same package that
changes the data in a fashion that is not desirable from a business perspective.

In other programming languages this is usually prevented by more granular scoping. You could, for example, use classes 
and create private member variables to **encapsulate** the data.

You have two options to deal with this problem: 

1. Trust that no one is going to violate the integrity of any data stored. 
2. Organize your code in such a way that each package only contains a minimal amount of code.

You can, of course, go with option 1., but I've never seen that go right. There's always that one colleague who is in a
hurry and implements something without thinking. Scoping is there to make the bad things hard and the good things easy.
This is called [defensive programming](https://en.wikipedia.org/wiki/Defensive_programming).

In Go defensive programming means you have to create a *lot* of directories. And I mean a lot. And you thought Java had
too many files and directories...

## The lack of immutability

Another useful tool in defensive programming is *immutability*. Immutable data structures prevent modifications to
data structures after they have been created, they can only be copied. While this is not as efficient in terms of 
performance it is also desirable to prevent accidental side effects.

Imagine an HTTP request struct: the first layer of your application creates it and then passes it down through several
modules. If the request struct was passed down as a pointer any layer modifying the request will modify it globally,
leading to a potential side effect in the top layer.

Even if you don't use pointers and pass the struct by value [slices](https://blog.golang.org/slices-intro) are still 
mutable data structures.

Go seems to heavily prioritize performance over avoiding potential bugs. There is no in-language way to create safe data
transfers at module boundaries apart from [third party deep copy libraries](https://github.com/jinzhu/copier). (There
are a lot of buggy ones too!)

In other words, barring the use of deep copy libraries, a developer **must** know what happens to the data throughout 
the application to be sure there are no unintended side effects.

## The lack of generics

[Generics](https://en.wikipedia.org/wiki/Generic_programming) are a convenient way of creating reusable code. Let's say
we want to build a *tree*. In Java this could look as follows:

```java
tree = new TreeNode()
tree.addChild(
    new TreeNode("Hello world!")
)
//...
auto data = tree.getChild(0).getData()
```

If we write the code like this the `data` variable will have the type `Object` with no specific information. We won't
have code completion for the fact that it is actually a string. We will need to *know* and cast it:

```java
auto data = (String)tree.getChild(0).getData()
```

If the data included is *not* a string this will result in an error. To work around this issue we can use *generics* to
give a type to the data included:

```java
tree = new TreeNode<String>()
tree.addChild(
    new TreeNode<String>("Hello world!")
)
//...
auto data = tree.getChild(0).getData()
```

In this case `data` will be a string without any further magic and we can be sure that there are only strings in the
tree.

Now, this is a feature that's sorely lacking from Go. We constantly have to keep casting to the data type we believe or
hope will be returned.

## OOP (the bad parts)

Since we talked about trees, let's take a look at how we would implement a tree node in Go.

```go
package tree

type TreeNode struct {
    children []treeNode
    data interface{}
}

func New(
    data interface{},
) *TreeNode {
    return &TreeNode{}
}

func (treeNode *TreeNode) AddChild(
    child *TreeNode,
) {
    treeNode.children = append(
        treeNode.children,
        child,
    )
}

func (treeNode *TreeNode) GetChild(
    childIndex int,
) *TreeNode {
    return treeNode.children[childIndex]
}
```

Yes, it's lacking error handling, but you get the idea. We don't have a *class* per se, but we do have this weird
construct called a **receiver** which is this part: `(treeNode * TreeNode)`.
Receivers are basically what the keyword `this` or `self` would be in other OOP languages.

In Go you would use the `TreeNode` like this:

```go
tree := New(data here)
tree.AddChild(New(data here))
```

You can even implement interfaces:

```
type TreeNodeInterface interface {
    AddChild(child TreeNode)
    GetChild(childIndex int) TreeNode
}
```

Quite simple and the interface above already implements this interface. No special keywords required, which is not
exactly helpful when diving into new code. IDEs like Goland help you with code navigation, but it can be quite hard
to follow which implementation is where. Furthermore, if you fail to implement even one function required for an
interface the code navigation doesn't work anymore.

It is also worth mentioning that the lack of inheritance, and the lack of generics together makes it very hard to write
reusable code.  

## No enums

The issues before were just quirky and possibly arise out of the way the language was built. Now we get to the 
downright silly. When I was implementing my [SSH server](https://github.com/janoszen/containerssh) I ran across a piece
of code that read like this:

```go
newChannel.Reject(reason, message) error
```

The reason was of the type `channel.RejectionReason`. And what do you think that type was? Let's use the powers of
the IDE to figure it out. (You shouldn't have to do this.)

```go
// RejectionReason is an enumeration
// used when rejecting channel creation
// requests. See RFC 4254, section 5.1.
type RejectionReason uint32
```

Cool. So it's a 32 bit unsigned integer. What values can it take? No clue. Is it going to check if I send it an invalid
code? Nope. There are no *enums* in Go. The possible values are defined separately:

```go
const (
	Prohibited RejectionReason = iota + 1
	ConnectionFailed
	UnknownChannelType
	ResourceShortage
)
```

These values are in *no relation to the above type definition*. No IDE in the world can give you code completion for
that and you *have* to dive into the libary you are using to figure this out.

## Package management

For a long time package management in Go was absolutely horrible. Thankfully, since 1.11
[Go modules](https://blog.golang.org/using-go-modules) are a thing, but dependency management is a very long way from
where other languages are.

As a major issue they use git as a package management tool. Packages generally don't contain any meta information
about their author, license, version number, etc. There is no code freeze after a version has been published absolutely
breaking the assumption that a version isn't going to change later. There is no clean way to comply with the Apache
license requirement to include the `NOTICE` file in your final builds in any reasonable fashion. Yes, I know, most
don't give a *flying flamingo*, but really, you should.

## The hype

Finally, the stupidest of all: the hype. Go is an awesome language if you want to build system-level stuff where
even micro optimization can bring a clear benefit. It was very clearly built with execution speed in mind over 
everything else. Even code maintainability.

Every larger Go project I came across is a horrible mess with more dependencies than your average Javascript framework.
The code itself puts Italy to shame when it comes to making spaghetti. You have to focus on tiny implementation
details rather than the grand concept.

Go is not well suited for applications with heavy business logic. Or webshops. Or 99% of the projects you come across
as an average developer. You are not Google and you don't have Google-like performance problems. (Or if you are 
what the hell are you doing reading my blog anyway?)

## Conclusion

Go is certainly [an awesome tool](/blog/go-is-awesome) for system-level development, but it comes at a cost in terms of
cognitive load. I definitely wouldn't want to write something with a heavy business logic in it, but I am struggling to
find a better tool for writing high performance system tools or utilities with. 
