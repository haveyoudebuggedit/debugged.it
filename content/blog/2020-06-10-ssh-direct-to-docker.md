---
categories:
- Technology
slug: "ssh-direct-to-docker"
guid: https://pasztor.at/blog/"ssh-direct-to-docker"/
date: "2020-06-10T00:00:00Z"
publishDate: "2020-06-10T00:00:00Z"
summary: Let's build an SSH server in Go that launches Docker containers for each
  session!
images:
- posts/ssh-direct-to-docker.png
preview: posts/ssh-direct-to-docker.jpg
tags:
- Golang
- Containers
authors:
- janos
title: Building a custom SSH server for fun and... containers!
---

During my career I had several projects in the web-hosting business. Partially driven by engineering pride I always
wanted to give customers the maximum amount of freedom to do what they have to do. As early as 2009 the team I was
leading provided full on SSH access to maintain their sites. That was at a time when even FTP encryption was still
a rarity. Providing SSH access, however, was not without its challenges.

{{% tip %}}
**In a hurry?** I have written a [fully functional SSH microservice](https://github.com/janoszen/containerssh) that launches containers. The sample code for [this post is also available on GitHub](https://github.com/janoszen/minicontainerssh).
{{% /tip %}}

![A demonstration of the SSH server in action.](posts/ssh-in-action.gif "The SSH server in action.")

Traditionally web hosting environments that offer SSH did so by providing a per-site or per-customer environment.
These environments are separated from each other by the virtue of creating separate Linux users for each environment.
This approach works well if each site has its own environment, but it presents a danger when a customer has multiple
sites hosted in the same environment. If any one of these sites is infected there is a risk of cross-contamination.

Both approaches have a significant drawback: there is no option to allow a user to access a specific set of sites from
a single SSH user. It's all or nothing. Either the user sees all sites in an environment or just a single one. There is
no option to make the user see a specific set of sites. 

![User A can access websites A and B, User B can access websites B and C](posts/ssh-multi-access.svg "Ideally this kind of access should be possible via SSH.")

## Containers to the rescue 

Since the traditional approach uses Linux system users to separate sites from each other creating an SSH user that
can access multiple sites is almost impossible simply due to permission problems. What if we took to a
*&ldquo;new&rdquo;* technology to solve the problem?

You, of course, know where I'm going with this, but it's nothing new. Containers have been around for 15 years on Linux
and even longer on other operating systems. However, the have only recently reached a stage of wide-spread adoption and
general usability.

![Animation: first an SSH client is started. This launches a container. Then a second SSH is started which launches a second container and so on.](posts/ssh-docker-anim.gif "Here's the plan.")

How about we put each site in a *container* instead of creating separate users? The PHP, Python, or what have you
website engine runs in a container, and the data directory the website is located in is *mounted* for that container?

If you are a user of [Docker](https://www.docker.com/) or [Kubernetes](https://kubernetes.io/) this may seem trivial,
but it's important to stress how much technology has evolved in a short timespan. Back in 2009-2010 mounting around 10k
mount points meant that a server reboot took a whopping 20-30 minutes.

Technology aside, how does this solve our SSH problem? Well, you see, with this approach sites are separated by
containers and not Linux system users. All sites are running as *the same user* from the perspective of the host
operating system, so all files will be owned by the same user.

This enables us to create a **special container** for each user. This special container mounts only the sites the user
in question has access to. When the user opens an SSH session they should land in this container, without access to
other. Since the actual runtime environment is running in a different container the user also can't accidentally kill
the running webserver. Nice and clean. 

## Hacking the SSH daemon

For a period of 6-7 years this approach above was merely a theory. While I have written a concept that used
[OpenVZ](https://openvz.org/) containers (or Virtual Environment as they called) in 2011 and that concept was later
realized, SSH access remained a difficult topic.

Python was a prime contender for writing an SSH server that would proxy connections based on what username the user
entered but that project never got off the ground. If I were to use Python for this purpose nowadays I would look into
libraries like [paramiko](https://github.com/paramiko/paramiko).

## Enter: Apache Mina

After a few years pause I had another brush with the web-hosting business in 2017, and I was determined to solve
this problem. Luckily by that time I had the good fortune to come across some amazing Java developers who inspired me
to take a look at [Apache Mina](https://mina.apache.org/).
 
Apache Mina offers the ability to create an SSH (and FTP) server in pure Java. The Mina SSHD allows for the definition
of handlers and hooks for user authentication and the actual execution of the requested shell. It does so without
needing users on the operating system level. The whole connection handling can be done entirely in Java. 

This offered me a chance: if I could marry SSH server with the
[Docker API](https://docs.docker.com/engine/api/latest/) I could potentially take everything that is coming through
the SSH connection and send it directly to the Docker API and vice versa. Effectively, I would be bypassing the regular
SSH daemon, and the potential configuration mishaps that could lead to a container escape.

To my surprise that's exactly how it worked. Implementing the `PasswordAuthenticator` and `PublickeyAuthenticator`
allowed me to provide custom authentication for users, while writing the `DockerizedCommand` class implementing
the `Command` interface allowed me to launch containers for the Docker container. The
[ContainerAttach](https://docs.docker.com/engine/api/v1.40/#operation/ContainerAttach) operation allowed me to take any
data coming through the SSH channel and send it to the container engine and vice versa.

However, the Java implementation was not without its problems. The Docker libraries that existed at the time were
incomplete and used a different async IO model than Apache Mina. When a user launched an SFTP session to download data
my SSH server would pull the data from the Docker API as fast as possible and push it into the SSH channel. It would
then sit there and wait for the user to download the data, consuming memory in the process.

In some rare cases users exhausted the 4 GB of RAM present on the server which lead to a crash. Adding more RAM helped, of course, but the situation was far from ideal.

## Rewriting it in Go

When I recently started learning Go I have discovered that the extended standard library contains a
[fully functional SSH library](https://godoc.org/golang.org/x/crypto/ssh). This presented the perfect learning project
for me: reimplement the SSH server in Go.

{{% tip %}}
**Note:** You can grab [the code in this post from GitHub](https://github.com/janoszen/minicontainerssh). Please note that it emphasizes learning over structure. If you would like to see a more production-ready version take a look at my [ContainerSSH project](https://github.com/janoszen/containerssh).
{{% /tip %}}

First we start with a nice and easy TCP server. This one will `Listen` on a certain IP and port for connections:

```golang
func main() {
    listener, err := net.Listen(
        "tcp",
        "0.0.0.0:2222",
    );
    if err != nil {
        log.Fatalf(
            "Failed to listen on port 2222 (%s)",
            err,
        )
    }
    log.Printf("Listening on 0.0.0.0:2222")
}
```

Easy, right? Now, the `Listen` call only opens a listening socket but does not accept connections. That's what
the `Accept` call is for. The `Accept` call will block the execution until a new connection is coming in.

```golang
func main() {
    //...
    for {
        tcpConn, err := listener.Accept()
        if err != nil {
            log.Printf(
                "Failed to accept (%s)",
                err,
            )
            //Continue with the next loop
            continue
        }
    }
}
```

### Establishing the SSH connection

So far so good, we have a working TCP connection now but no SSH protocol decoding is taking place.
That's where the next bit comes in: initializing the SSH connection. Before we do that we will need to build
an SSH configuration. Let's get that out of the way. First let's add a password authentication method:

```golang
func main() {
    //...
    sshConfig := &ssh.ServerConfig{}

    // region SSH authentication
    sshConfig.PasswordCallback = func(
        conn ssh.ConnMetadata,
        password []byte,
    ) (
        *ssh.Permissions,
        error,
    ) {
        if conn.User() == "foo" &&
            string(password) == "bar" {
            return &ssh.Permissions{}, nil
        } else {
            return nil, fmt.Errorf(
                "authentication failed",
            )
        }
    }
    //endregion

    for {
        //The previously written Accept code here
    }
}
```

The `PasswordCallback` and the `PublicKeyCallback` allow for password and public key authentication from a database
instead of the system. Next we have to create a host key. On Linux systems this can be done using `ssh-keygen -t rsa`.
The loading procedure for the keys looks like this:

```golang
func main() {
    //...
    sshConfig := &ssh.ServerConfig{}
    //...

    // region Host key
    hostKeyData, err := ioutil.ReadFile(
        "ssh_host_rsa_key",
    )
    if err != nil {
        log.Fatalf(
            "failed to load host key (%s)",
            err,
        )
    }
    signer, err := ssh.ParsePrivateKey(
        hostKeyData,
    )
    if err != nil {
        log.Fatalf(
            "failed to parse host key (%s)",
            err,
        )
    }
    sshConfig.AddHostKey(signer)
    // endregion

    for {
        //The previously written Accept code here
    }
}
```

Now that we have that sorted we can actually accept the SSH connection:

```golang
func main() {
    //...
    for {
        //...
        sshConn, chans, reqs, err := ssh.NewServerConn(
            tcpConn,
            sshConfig,
        )
        if err != nil {
            log.Printf(
                "handshake failed (%s)",
                err,
            )
            continue
        }
    }
}
```

This will perform the SSH handshake and establish a secure connection. It will return a number of
things:

![A single SSH connection can contain multiple SSH channels and global requests can be sent directly over the connection. Each channel transports data and channel requests in both directions.](posts/ssh.svg "SSH connection anatomy")

- `sshConn` is the actual SSH connection.
- `chans` is a Go channel where new SSH channels come in. An SSH connection can have multiple SSH channels
  to handle different kinds of parallel data transfers in the same SSH connection.
- `reqs` is a Go channel where requests come in. Requests are ways for the client to request a change
  to something.
- `err` is, of course, the error if any happened.

{{% tip %}}
**Go channels:** Go has a very efficient multiprogramming model called *goroutines*. Go channels are 
a way to send data between these goroutines. You can imagine these like an in-application message queue. If you want
to learn more check out [Go by Example: Channels](https://gobyexample.com/channels)
{{% /tip %}}

At this point we have an SSH connection established, and we need to go to work on requests and channels. First of all, we
are going to reject all global requests. Global requests would be used to request, for example, port forwarding, which
we don't support.

```golang
func main() {
    //...
    for {
        //...
        //Reject all global requests.
        //Run this in a goroutine so it
        //doesn't block.
        go ssh.DiscardRequests(reqs)
    }
}
```

Next we need to handle the incoming channels. The waiting for channels will be handled in a goroutine, so we don't block
the main program execution. When a channel comes in we will then create a further goroutine to handle that specific
channel.

```golang
func main() {
    //...
    for {
        //...
        //Reject all global requests.
        //Run this in a goroutine so it
        //doesn't block.
        go handleChannels(sshConn, chans)
    }
}

func handleChannels(
    conn *ssh.ServerConn,
    chans <-chan ssh.NewChannel,
) {
    for newChannel := range chans {
        go handleChannel(conn, newChannel)
    }
}

func handleChannel(conn *ssh.ServerConn, newChannel ssh.NewChannel) {
    //Handle new channel here
}
```

### Handling SSH channels

In the `handleChannel` function we have two options: either we accept the channel or we reject
it. First of all, our SSH server will only support the `session` channel type so let's reject
everything else:

```golang
//...
func handleChannel(
    conn *ssh.ServerConn,
    newChannel ssh.NewChannel,
) {
    if t := newChannel.ChannelType();
        t != "session" {
        _ = newChannel.Reject(
            ssh.UnknownChannelType,
            fmt.Sprintf(
                "unknown channel type: %s",
                t,
            ),
        )
        return
    }
}
```

Next up we need to create a connection to the Docker engine. We will do this using the `github.com/docker/docker/client`
package:

```golang
//...
func handleChannel(
    conn *ssh.ServerConn,
    newChannel ssh.NewChannel,
) {
    //...
    docker, err := client.NewClient(
        "tcp://127.0.0.1:2375",
        "",
        nil,
        make(map[string]string),
    )
    if err != nil {
        _ = newChannel.Reject(
            ssh.ConnectionFailed,
            fmt.Sprintf(
                "error contacting backend (%s)",
                err,
            ),
        )
        return
    }
}
```

So far so good. Now we need to define a couple of variables that we will use later on:

```golang
//...
type channelProperties struct {
    // Allocate pseudo-terminal for
    // interactive sessions.
    pty bool
    // Store the container ID
    // once it is started.
    containerId string
    // Environment variables passed
    // from the SSH session.
    env map[string]string
    // Horizontal screen size
    cols uint
    // Vertical screen size
    rows uint
    // Context required by the Docker client.
    ctx context.Context
    // Docker client
    docker *client.Client
}

func handleChannel(
    conn *ssh.ServerConn,
    newChannel ssh.NewChannel,
) {
    //...
    channelProps := &channelProperties{
        pty:         false,
        containerId: "",
        env:         map[string]string{},
        cols:        80,
        rows:        25,
        ctx:         context.Background(),
        docker:      docker,
    }
}
```

Finally, let's accept the channel. If the channel accept fails we will close the Docker
connection and return from the channel handling.

```golang
//...
func handleChannel(
    conn *ssh.ServerConn,
    newChannel ssh.NewChannel,
) {
    //...
    connection, requests, err :=
        newChannel.Accept()
    if err != nil {
        log.Printf(
            "could not accept channel (%s)",
            err,
        )
        err := docker.Close()
        if err != nil {
            log.Printf(
                "error while closing (%s)",
                err,
            )
        }
        return
    }
}
```

### Channel-specific requests

Having established a channel the client can now send data and channel-specific requests. Channel-specific
requests can be any number of things including custom ones. We will discuss the following:

`env`
: to set environment variables.

`pty-req`
: to allocate a pseudo-terminal for interactive sessions. (You need this for moving around with the cursor.)

`window-change`
: to change the window size.

`shell`
: to execute the default shell. 

Due to complexity we will not cover the following request types. You can look at
[the source code of ContainerSSH](https://github.com/janoszen/containerssh) for details.

`exec`
: to execute a custom program.

`subsystem`
: to launch a named subsystem, for example SFTP.

`signal`
: to send a signal to the process.

So let's work on implementing our channel handler:

```golang
//...
func handleChannel(
    conn *ssh.ServerConn,
    newChannel ssh.NewChannel,
) {
    //...
    removeContainer := func() {
        if channelProps.containerId != "" {
            //Remove container
            removeOptions :=
                types.ContainerRemoveOptions{
                    Force: true,
                }
            err := docker.ContainerRemove(
                channelProps.ctx,
                channelProps.containerId,
                removeOptions,
            )
            if err != nil {
                log.Printf(
                    "error while removing (%s)",
                    err,
                )
            }
            channelProps.containerId = ""
        }
    }
    closeConnections := func() {
        removeContainer()
        //Close Docker connection
        err = docker.Close()
        if err != nil {
            log.Printf(
                "error while closing Docker (%s)",
                err,
            )
        }
        //Close SSH connection
        err := conn.Close()
        if err != nil {
            log.Printf(
                "error while closing SSH (%s)",
                err,
            )
        }
    }

    go func() {
        for req := range requests {
            reply := func(
                success bool,
                message []byte,
            ) {
                if req.WantReply {
                    err := req.Reply(
                        success,
                        message,
                    )
                    if err != nil {
                        closeConnections()
                    }
                }
            }

            handleRequest(
                channel,
                req,
                reply,
                closeConnections,
                removeContainer,
                channelProps
            )
        }
    }()
}
```

We are basically creating *yet another* goroutine to deal with channel requests, and the handling
will be done in a separate function called `handleRequest`. The request handler has the option to reply
to the request if needed. We also define a function called `closeConnections` that we will call if
something goes wrong, or if the connections need to be closed naturally.

### Implementing the request handler

As a final piece of the puzzle let's implement the `handleRequest` function. First let's implement
a default type that rejects all requests:

```golang
//...
type envRequestMsg struct {
    Name  string
    Value string
}

func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    default:
        reply(
            false,
            []byte(fmt.Sprintf(
                "unsupported request type (%s)",
                req.Type,
            )),
        )
    }
}
```

Quite simple so let's move on to the `env` request type. This one is used to set environment
variables:

```golang
//...
type envRequestMsg struct {
    Name  string
    Value string
}

func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    case "env":
        if channelProps.containerId != "" {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "cannot set env variables",
                ),
            ))
            return
        }
        request := envRequestMsg{}
        err := ssh.Unmarshal(req.Payload, request)
        if err != nil {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "invalid payload (%s)",
                    err,
                )),
            )
            return
        }
        channelProps.env[request.Name] =
            request.Value
    default:
        //...
    }
}
```

As you can see we declare an `envRequestMsg` struct. This struct is the format of the `Payload` part of the
request and will be decoded using the `ssh.Unmarshal()` function call. The environment variable received
will be stored in the `env` variable.

Since that was pretty painless let's implement the `pty-req` type to handle interactive terminals:

```golang
//...
func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    case "env":
        //...
    case "pty-req":
        if channelProps.containerId != "" {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "cannot set pty after shell",
                )),
            )
            return
        }
        channelProps.pty = true
    default:
        //...
    }
}
```

Even simpler, we are simply setting a boolean when a PTY request is coming in. Before we dive into the ugly bits let's
quickly implement the `window-change` handler:

```golang
type windowChangeRequestMsg struct {
    Columns uint32
    Rows    uint32
    Width   uint32
    Height  uint32
}

func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    case "env":
        //...
    case "pty-req":
        //...
    case "window-change":
        request := windowChangeRequestMsg{}
        err := ssh.Unmarshal(req.Payload, request)
        if err != nil {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "invalid payload (%s)",
                    err,
                )),
            )
            return
        }
        channelProps.cols = uint(request.Columns)
        channelProps.rows = uint(request.Rows)
        if channelProps.containerId != "" {
            err = channelProps.
                docker.
                ContainerResize(
                    channelProps.ctx,
                    channelProps.containerId,
                    types.ResizeOptions{
                        Height: channelProps.rows,
                        Width:  channelProps.cols,
                    },
                )
            if err != nil {
                reply(
                    false,
                    []byte(fmt.Sprintf(
                        "failed to set wnd (%s)",
                        err,
                    )),
                )
                return
            }
        }
    default:
        //...
    }
}
```

As you can see we are already starting to interact with the Docker API. When the client (e.g. PuTTY) window
size changes we also send the new dimensions to the container. This is useful in cases where a software like
[midnight commander](https://midnight-commander.org/) is running in PTY mode as it needs the correct dimensions
to scale to the full window.

## Launching the container

We have one last task before we can take our brand new SSH server for a spin: implement the `shell` request.
This request will launch a Docker container and connect the container to the SSH input/output. This will
enable us to actually use SSH.

As a first step of launching the container we will need to pull the image we want to run:

```golang
type windowChangeRequestMsg struct {
    Columns uint32
    Rows    uint32
    Width   uint32
    Height  uint32
}

func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    case "env":
        //...
    case "pty-req":
        //...
    case "window-change":
        //...
    case "shell":
        if channelProps.containerId != "" {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "cannot launch a second shell"
                )),
            )
            break
        }
        pullReader, err := channelProps.
            docker.
            ImagePull(
                channelProps.ctx,
                "docker.io/library/busybox",
                types.ImagePullOptions{},
            )
        if err != nil {
            reply( 
                false,
                []byte(fmt.Sprintf(
                    "could not pull busybox (%s)",
                    err,
                )),
            )
            return
        }
        _, err = ioutil.ReadAll(pullReader)
        if err != nil {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "could not pull busybox (%s)",
                    err,
                )),
            )
            return
        }
        err = pullReader.Close()
        if err != nil {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "could not pull busybox (%s)",
                    err,
                )),
            )
            return
        }
    default:
        //...
    }
}
```

Now we can be sure the intended target image is available locally, and we can create the container:

```golang
func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    //...
    case "shell":
        //...
		var env []string
		for key, value := range channelProps.env {
			env = append(
                env,
                fmt.Sprintf(
                    "%s=%s",
                    key,
                    value,
                ),
            )
		}
		body, err := channelProps.
            docker.
            ContainerCreate(
                channelProps.ctx,
                &container.Config{
                    Image: "busybox",
                    AttachStdout: true,
                    AttachStderr: true,
                    AttachStdin: true,
                    Tty: channelProps.pty,
                    StdinOnce: true,
                    OpenStdin: true,
                    Env: env,
                },
                &container.HostConfig{},
                &network.NetworkingConfig{},
                "",
            )
        if err != nil {
            reply(
                false,
                []byte(fmt.Sprintf(
                    "failed to launch (%s)",
                    err,
                )),
            )
            return
        }
        channelProps.containerId = body.ID
    default:
        //...
    }
}
```

The container is created so let's prepare the attach before we start it:

```go
func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    //...
    case "shell":
        //...
        attachResult, err := channelProps.
            docker.
            ContainerAttach(
                channelProps.ctx,
                channelProps.containerId,
                types.ContainerAttachOptions{
                    Logs:   true,
                    Stdin:  true,
                    Stderr: true,
                    Stdout: true,
                    Stream: true,
                },
            )
        if err != nil {
            removeContainer()
            reply(
                false,
                []byte(fmt.Sprintf(
                    "failed to attach (%s)",
                    err,
                )),
            )
            return
        }
    default:
        //...
    }
}
```

When that's done starting it is a simple API call away:

```go
func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    //...
    case "shell":
        //...
        err = channelProps.docker.ContainerStart(
            channelProps.ctx,
            channelProps.containerId,
            types.ContainerStartOptions{},
        )
        if err != nil {
            removeContainer()
            reply(
                false,
                []byte(fmt.Sprintf(
                    "failed to launch (%s)",
                    err,
                )),
            )
            return
        }
    default:
        //...
    }
}
```

Before we start transferring data from and to the container we still have to set the window size from any previous
`window-change` requests:

```go
func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    //...
    case "shell":
        //...
        err = channelProps.
            docker.
            ContainerResize(
                channelProps.ctx,
                channelProps.containerId,
                types.ResizeOptions{
                    Height: channelProps.rows,
                    Width:  channelProps.cols,
                },
            )
        if err != nil {
            removeContainer()
            reply(
                false,
                []byte(fmt.Sprintf(
                    "failed to resize (%s)",
                    err,
                )),
            )
            return
        }
    default:
        //...
    }
}
```

And that's it! Container is running and attached, window is of the correct size, environment variables are set, let's
have a &ldquo;go&rdquo; at pushing data from and to the container.

```go
func handleRequest(
    channel ssh.Channel,
    req *ssh.Request,
    reply func(success bool, message []byte),
    closeConnections func(),
    removeContainer func(),
    channelProps * channelProperties,
) {
    switch req.Type {
    //...
    case "shell":
        //...
        var once sync.Once
        if channelProps.pty {
            go func() {
                _, _ = io.Copy(
                    channel,
                    attachResult.Reader,
                )
                once.Do(closeConnections)
            }()
        } else {
            go func() {
                //Demultiplex Docker stream
            	//into stdout/stderr
                _, _ = stdcopy.StdCopy(
                    channel,
                    channel.Stderr(),
                    attachResult.Reader,
                )
                once.Do(closeConnections)
            }()
        }
        go func() {
            _, _ = io.Copy(
                attachResult.Conn,
                channel,
            )
            once.Do(closeConnections)
        }()
    default:
        //...
    }
}
```

This one bears a little bit of an explanation. If the container is run in interactive mode everything coming
from the application comes over stdout in a binary form. If, however, the application is run non-interactively the SSH
channel expects the data to come via two separate streams on stdout and stderr. Docker on the other hand will still
utilize a multiplexed return format, so we need to use `stdcopy.StdCopy` to demultiplex that stream into two separate
ones.

Finally, when one of the streams ends we use the `closeConnection` function we defined previously to shut
everything down.

## Testing it

The total code weighs at a little over 300 lines of code, so it's time to take our brand new SSH server for a spin. On
the first terminal:

```bash
$ go run main.go
2020/06/09 15:50:31 Listening on 0.0.0.0:2222
```

And on a second one:

```bash
$ ssh localhost -l foo -p 2222
foo@localhost's password:  <--- enter "bar" here
/ #
```

That's it! We have a running SSH server!

As mentioned in the beginning you can grab the full code [from GitHub](https://github.com/janoszen/minicontainerssh) and
if you want a more complete implementation with `exec`, `signal` and SFTP as well as a more structured codebase take a
look at my [ContainerSSH project](https://github.com/janoszen/containerssh).

## Adapting it for Kubernetes

So far we have only talked about Docker, but Kubernetes is definitely the winner of the container orchestrator race.
Thankfully, the Kubernetes API also has the ability to attach to a pod similar to how Docker works. I'll leave it up
you, the reader to play with the [Kubernetes SDK](https://github.com/kubernetes/client-go) and figuring out how to
implement it into your SSH server.

## Is it production grade?

When talking about SSH security undoubtedly comes to mind. Is an SSH implementation in Go secure enough? Will it be
properly maintained?

The truth is, I don't know. When I was trying to set up the
[Mozilla recommend cipher suites](https://infosec.mozilla.org/guidelines/openssh) and
[I immediately encountered a crash](https://github.com/golang/go/issues/39397). I have now submitted a patch for this
issue and ultimately this crash did not cause a security issue but the fact that even basic configuration checks are
missing is a bit worrying.

The foundations are solid and the
[list of supported ciphers, key exchange algorithms and MACs](https://github.com/golang/crypto/blob/master/ssh/common.go#L29)
seem to be good enough to provide a secure connection, but my gut feeling is that this library needs to mature a bit to
be considered production ready.

## Use cases

Creating a custom SSH server that launches containers goes way beyond simple web hosting. Imagine a school where
students need to access temporary environments. Or a Linux / security education environment. These cases of firing up
workloads on demand is what the cloud and Kubernetes were basically made for, but there is no simple way to SSH into
them.
