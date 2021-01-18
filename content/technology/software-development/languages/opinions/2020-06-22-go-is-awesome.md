---
categories:
- Technology
slug: go-is-awesome
guid: https://pasztor.at/blog/go-is-awesome/
date: "2020-06-22T00:00:00Z"
publishDate: "2020-06-22T00:00:00Z"
summary: Go certainly caught a lot of attention. Let's look at the good parts!
images:
- posts/go-is-awesome.png
preview: posts/go-is-awesome.jpg
tags:
- Software Development
- Golang
title: Go is an awesome language
authors:
- janos
---

I have recently written an [SSH server that launches containers](https://github.com/janoszen/containerssh) in Go.
The project has certainly grown to quite a large size, and I have also sent a
[pull request to Golang itself](https://go-review.googlesource.com/c/crypto/+/236517) to fix a bug I found. After having
gathered substantially more experience than a `Hello world!`, I now feel confident to lay out the parts I really like
about the language. 

{{% tip %}}
**Do you want a different opinion?** Read my post about [why Go is terrible](/blog/go-is-terrible)!
{{% /tip %}}

## Cross-platform

One of the reasons Go caught my eye was its build system. The original promise of Java was that it would be cross-platform, but the fact that you needed to install a runtime was clearly a turnoff. Go, on the other hand, compiles to native binaries. On Windows, you will get an <code role="text" aria-label="E X E">.exe</code> file, on Linux an ELF binary and so on. What's more, unless you use [cgo](https://golang.org/cmd/cgo/) your Go program can live with *almost* no external dependencies. No need to install any <code role="text" aria-label="D L L">.dll</code> or <code role="text" aria-label="S O">.so</code> files, a Go program will [just work](https://www.youtube.com/watch?v=YPN0qhSyWy8) out of
the box.

It's a common misconception that Go programs will run entirely without external dependencies, but they get as close
as humanly possible. Some libraries, such as [<code role="text" aria-label="lib-c">libc</code>](https://en.wikipedia.org/wiki/C_standard_library), are still
required for a number of functions.

The fact that Go could be used to build true cross-platform binaries that work without installing a clunky runtime, 
such as with Java or Python, was my main draw to Go.

## Goroutines and channels

Once I got into Go a little more I realized how cool its handling of *concurrency* was. Traditionally you would
utilize either threads or separate processes to run multiple tasks concurrently (<abbr role="text" aria-label="for example">e.g.</abbr> Java, C, C++). Alternatively, you
would rely on cooperative multitasking (<abbr role="text" aria-label="for example">e.g.</abbr> Javascript) to the same effect.

With threads and processes every switch the operating system has to do incurs a time penalty. This is called a 
[context switch](https://en.wikipedia.org/wiki/Context_switch). In other words, a careless programmer who uses tons of
threads will have performance problems down the line.

Cooperative multitasking on the other hand will run on a single thread. Whenever one task has to wait for something a
different task will run. If a task hogs the CPU other tasks will be starved.

Go combines the two in an ingenious way. Let's take the following example:

```go
func main() {
    go someOtherFunction()
}
```

Notice the `go` keyword. By using this keyword `someOtherFunction()` runs in a *goroutine*. Imagine the way Go deals
with concurrency as a pool of threads. Whenever you run a goroutine it will run in one of these threads.  This way Go
optimizes the use of threads for performance.

To facilitate data transfer between goroutines Go introduces *channels*, which are in-application message queues to
send data.

```go
func main() {
    chan done <- bool

    go func() {
        time.Sleep(2 * time.Second)
        done <- true
    }()

    //This will wait until the goroutine finishes
    <- done
}
```

As you can see from the code above the `<- channelname` will block the execution of the current goroutine until there
is data available and makes it extremely easy to do concurrent programming.

If you are interested in more details take a look at [channels](https://gobyexample.com/channels),
[contexts](https://gobyexample.com/context), and [mutexes](https://gobyexample.com/mutexes).

## Pointers, Defer, and Garbage Collection

When you think of [pointers](https://gobyexample.com/pointers) you first think of C or C++. Usually that memory is not
a pleasant one.

In Go pointers are more like *references*. Instead of always copying the data in a variable a pointer, well, points
to the original piece of memory. It does not matter how many times you pass on the variable containing a pointer any
modification will always change the original.

Let's look at an example: 

```go
someVar := &someStruct{}
```

The variable now contains a *pointer* to the struct. As it is passed along it always refers to the same space of memory
no matter how many times you copy the pointer.

However, unlike in C pointers, Go pointers are automatically garbage collected once they are no longer needed. You
don't need to worry about free-after-use or buffer overflow vulnerabilities, those are not a thing in Go. Which is
awesome.

Furthermore, you also have the `defer` statement to help you with cleanup after a function. Consider the following
function:

```go
func foo() error {
    close := func() {
        // Do something to
    	// clean up stuff
    }
    err := doSomething()
    if err != nil {
        close()
        return err
    }
    // Do something else
    close()
}
```

As you can see we repeated the `close()` call two times in this function. If we had more exit paths from this function
we would need to duplicate the `close()` call for each exit path.

The `defer` statement helps with exactly this problem:

```go
func foo() error {
    close := func() {
        // Do something to
    	// clean up stuff
    }
    defer close()

    err := doSomething()
    if err != nil {
        return err
    }
    // Do something else
}
``` 

That's it! The `defer` statement runs the `close()` function for every exit path of our function!

## Multiple return variables

It's a seemingly trivial thing to implement, but it's rather rare among programming languages.

```go
sshConn, chans, reqs, err := ssh.NewServerConn(
    tcpConn,
    config,
)
```

What's not to love?

## OOP (the good parts)

Although Go does not have a `class` construct you can still write object-oriented code.

Let's say you have the following Java code:

```java
class TreeNode {
    private List<TreeNode> nodes =
        new ArrayList<>();

    public void addChild(child TreeNode) {
        nodes.add(child)
    }
}
```

In Go a similar code would look like this:

```go
type TreeNode struct {
    children []treeNode
}

func New() *TreeNode {
    return &TreeNode()
}

func (treeNode * TreeNode) AddChild(child * TreeNode) {
    treeNode.children = append(
        treeNode.children,
        child,
    )
}
```

Go calls the `(treeNode * TreeNode)` part a *receiver*. Receivers in Go can work with any data type and functions very 
similar to the `this` keyword in other languages.

## Slices

Go, like many other low level languages, implements arrays as fixed-size lists of items. Their size cannot be changed
after they are created.

Slices, on the other hand, are a trick to make them dynamic. When a slice is *full* Go creates a new, larger copy of the
slice. Go optimizes the process in such a way that there are as few copy processes as possible.

Furthermore, Go slices also have a neat feature of creating *subarrays* that do not take up extra memory. These *slices*
of the original will reference the well, *slice* of the original. If you change data in the slice it will also change in
the original.

```go
import "fmt"

func main() {
	data := []string{"a", "b", "c", "d"}
	d := data[2:3]
	// Will print [c]
	fmt.Printf("%v", d)
	d[0] = "f"
	//Will print [a b f d]
	fmt.Printf("%v", data)
}
```

If you fancy a deeper look head on over to [Go by Example](https://gobyexample.com/slices).

## Libraries

One of the reasons speaking for Go is the copious amount of libraries. SSH client and server library?
[Covered.](https://godoc.org/golang.org/x/crypto/ssh) SDK for AWS? [Done.](https://aws.amazon.com/sdk-for-go/)
GitHub? [Of course.](https://github.com/google/go-github) Let's try something very rare... how about a FastCGI
protocol implementation? [Sure, why not.](https://golang.org/pkg/net/http/fcgi/)

I could keep going, but you get the picture. The popularity of Go has certainly helped the ecosystem.

## Tooling

The ubiquitous amount of things built for Go also show themselves in the amount of tooling available. You have
everything from [automatic code formatting](https://blog.golang.org/gofmt), [testing](https://golang.org/pkg/testing/),
to a [full-on release tool](https://goreleaser.com/). There are plenty of tools for just about anything. 

## Conclusion

Go certainly has [its downsides](/blog/go-is-terrible) when it comes to code organization. It is, however, uniquely suited
for low level, high performance software development for a wide range of tasks.
