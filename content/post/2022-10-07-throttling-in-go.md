---
title: "Throttling in Go"
date: 2022-10-07
tags: ["go", "howto", "code"]
---

Go language makes a good job at making hard things simple. It compiles the code so that it runs fast,
it comes with garbage collector so we don't have to deal with memory management, it has lightweight
goroutines and we don't have to think a lot about concurrency limits. And in majority of cases all those
language constructs combined with sane defaults will be good enough. Just like writing a web server.
Its almost never the case that we have to think about how many parallel requests we can handle.

But sometimes we need to have more control over our resources.

## Exploring [Restic]

Few years ago, while I was still a newbie in the go world, I was exploring internals of [Restic] -
a nice backup tool written in go. While doing so I learned a lot, but one thing caught my attention.
There was an upper limit of simultaneous upload to a remote storage such as an s3 server.

When I first saw [Restic] in action I thought that this must have been done using some upload queue,
maybe some upload scheduler or some other complicated machinery. But the implementation in [Restic]
turned out to be much simpler. Each upload is realized in a separate goroutine and there are many of
them spawned in parallel, much more than what is really necessary to saturate upload speed. But to avoid
unbounded number of parallel uploads, there's a throttling mechanism based on upload *tokens*. Those *tokens*
are [acquired][token_acquire] before sending data to the remote storage and [released][token_release]
once the upload is done. The number of upload *tokens* is limited constraining the number of parallel uploads.

[Restic]: https://restic.net/
[token_acquire]: https://github.com/restic/restic/blob/v0.5.0/src/restic/backend/s3/s3.go#L146
[token_release]: https://github.com/restic/restic/blob/v0.5.0/src/restic/backend/s3/s3.go#L184

Since I was a golang newbie back then, I didn't really understand the code when I first looked at it.
It was using channels so I thought that it is related to some communication between goroutines.
I only realized that this is a very neat throttling mechanism a bit later.

## Channels as token machines

How is it done? A dedicated go channel is used as a token disposal machine. To do some throttled operation,
one takes the token from that channel (`token := <- channel`) and gives it back once the operation is finished
(`channel <- token`). If there are no tokens left, the channel will naturally block the goroutine until someone
else returns previously acquired token back to the pool.

All of that can be done in just few lines of go code.

In this post I'd like to play a bit with this idea and show you how easy such throttlers can be implemented.

## The basic implementation

Let's start with some worker code. As we've discussed before the worker should first acquire the token
and give it back to the pool once the task is finished:

```go
func worker() {
    token := <- tokens                  // Grab the token, can block
    defer func() { tokens <- token }()  // Return the token

    // do the hard work...
}
```

Before we can use the `tokens` channel it must be initialized by filling it up with tokens:

```go
const tokensCount = 30
var tokens = make(chan struct{}, tokensCount)

func initTokens() {
    for i:=0; i<tokensCount; i++ {
        tokens <- struct{}{}
    }
}
```

We can also extract the throttler to a separate easy to use object:

```go
type Throttler struct {
    ch chan struct{}
}

func NewThrottler(max int) *Throttler {
    ch := make(chan struct{}, max)

    for i := 0; i < max; i++ {
        ch <- struct{}{}
    }

    return &Throttler{ch: ch}
}

func (c *Throttler) Acquire() func() {
    token := <-c.ch
    return func() { c.ch <- token }
}
```

and use it in our worker:

```go
const tokensCount = 30
var tokens = NewThrottler(tokensCount)

func worker() {
    defer tokens.Acquire()()    // Throttle

    // do the hard work...
}
```

You may notice that the `Acquire` method returns a `func()` object instead of the token.
That function will release the token back to the pool when executed.
That way the acquire and the release of the token are tightly coupled together
and we can both acquired the token and schedule its release upon
function exit in a nice one-liner.

## Reversed implementation

The implementation above can be interpreted as the channel being a set of idle tokens.
Worker takes the token from that set and gives it back once done with the job.

But we can also reverse the idea and use channel as a holder of tokens for busy workers.
Before doing the task, worker puts its token into the channel and takes it back
once the work is finished. Since the channel has the maximum capacity, once it is full,
more *busy* tokens will have to wait until someone takes one's token from the channel:

```go
type Throttler struct {
    ch chan struct{}
}

func NewThrottler(max int) *Throttler {
    ch := make(chan struct{}, max)
    return &Throttler{ch: ch}
}

func (c *Throttler) Acquire() func() {
    c.ch <- struct{}{}
    return func() { <-c.ch }
}
```

The biggest difference here is that the channel doesn't have to be populated with tokens
during initialization but otherwise there's not much of a difference between those two implementations.

## Old-style implementation

In previous examples the core of the throttler was a go channel. But the same can be implemented
using old-style synchronization primitives:

```go
type Throttler struct {
    cnt int
    m   sync.Mutex
    c   sync.Cond
}

func NewThrottler(max int) *Throttler {
    ret := &Throttler{
        cnt: max,
    }
    ret.c.L = &ret.m
    return ret
}

func (c *Throttler) Acquire() func() {
    c.m.Lock()
    defer c.m.Unlock()

    for c.cnt <= 0 {
        c.c.Wait()
    }
    c.cnt--

    return func() {
        c.m.Lock()
        defer c.m.Unlock()

        c.cnt++
        c.c.Signal()
    }
}
```

Much more complex implementation compared to the previous one, right?

When I first wrote it I assumed that there will be a huge performance boost because we replaced a *heavy*
channel object with a *lightweight* low-level synchronization mechanisms.

But after I run some benchmarks it turned out that this implementation is only between
12% to 19% faster than our original one. This does not sound like a huge difference to me.
Maybe in some very specific CPU-sensitive use cases it could be beneficial but I believe that
if each spent CPU cycle is making the difference then languages such as c or c++ (or even assembly)
would be a better fit there.

Putting performance difference aside, there's one more huge difference I'd like to point out.

## Cancellable waits

Our channel-based throttlers have one huge advantage over the one based on mutex.
They can easily be extended to also support cancellation through context.

Here's an example of a cancellable `Acquire`:

```go
func (c *Throttler) Acquire(ctx context.Context) (func(), error) {
    if ctx.Err() == nil {
        select {
        case <-c.ch:
            return func() { c.ch <- struct{}{} }, nil
        case <-ctx.Done():
            break
        }
    }
    return nil, ctx.Err()
}
```

If the context is cancelled for any reason, the token acquire will now simply return an error:

```go
func worker(ctx context.Context) error {
    releaseToken, err := tokens.AcquireCtx(ctx)
    if err != nil {
        return err
    }
    defer releaseToken()

    // do the hard work...
}
```

Why we would need such context-aware throttler? Let's take a simple http server as an example.
A http request can be cancelled at any time as a result of the client dropping the connection
or server shutdown. Waiting for the token in case the request is already cancelled
is most likely pointless and the request handler should return immediately.

## Low-level does not always mean more powerful

Such cancellable acquire can not be easily implemented with low-level mutexes and condition variables.
Locking a mutex or waiting on a condition in go can not be interrupted. This means that the channel-based
version is not only easier to write and understand but also handles much broader set of use-cases.

We could of course try experimenting with some complex implementation of a mutex-based throttler supporting cancellation.
But at this point I don't believe that we would gain much more with it.
Considering relatively small performance gains I'd rather stay with the channel-based implementation.

## Future improvements

I hope you liked my introduction to throttling in go. Of course I only scratched the surface of this topic
so stay tuned for future posts where I'll show few more tips and tricks.

## See you on [GitHub][go_limiters]

I've gathered various implementations of throttlers in [this github repository][go_limiters]. It's a well
tested (aiming at 100% test coverage) tiny go module that anybody can use. It also contains benchmarks
to check the performance difference between various implementations.

[go_limiters]: https://github.com/byo/go-limiters/tree/v0.1.0
