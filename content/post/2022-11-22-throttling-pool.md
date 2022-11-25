---
title: "Throttling Pool"
date: 2022-11-22
tags: ["go", "howto", "code"]
---

In my [last post][post-prev] I presented few techniques useful for limiting simultaneous operations in Go. Such throttlers are indeed a very neat tools to keep our resources under control. However we can be extended that idea to even better tool for resource control when we mix throttlers with object pools.

## Object pools in go

Even though golang has a very efficient memory management system, there are some cases where we would like to manage the memory by ourselves. There may be many reasons for that. One may want to ensure that the amount of allocated memory does not cross some predefined limit. It may also be the case that frequent allocations and deallocations of memory start causing issues due to a huge CPU cost. Those factors are especially important in more complex systems with tight memory budget such as databases. And it turns out that I had a chance to work with one such database called [immudb] for more than a year now.

[immudb]: https://immudb.io/

In `immudb` there are some [larger scratchpad objects][immudb-tx] used during the transaction commit that are [preallocated][immudb-txpool-init] when the database is loaded. Once a writer starts preparing a transaction, such *scratchpad* object is [taken from the pool][immudb-txalloc] to be [returned back to it][immudb-txrelease] once the commit finishes.

Sounds familiar? Looks almost identical to the technique used for throttling the number of simultaneous operations with tokens that I presented in the [previous post][post-prev]. But instead of fetching a token from the pool that is of no use, we can fetch a useful object. This mechanism is used by `immudb` to keep the number of simultaneous transactions [under control][immudb-txalloc-error]. Without such limit, it would be easy to *force* the database to use huge amount of resources (which some may refer to as a [DoS][dos] attack if for example the database runs out of memory).

The implementation in immudb is [based on a mutex][immudb-mutex-pool] whereas in this blog post I'll focus on a channel-based implementation that will be similar to what we've seen with token-based throttlers.

[dos]: https://en.wikipedia.org/wiki/Denial-of-service_attack

[immudb-tx]: https://github.com/codenotary/immudb/blob/v1.4.0/embedded/store/tx.go#L29
[immudb-txpool-init]: https://github.com/codenotary/immudb/blob/v1.4.0/embedded/store/immustore.go#L366
[immudb-txalloc]: https://github.com/codenotary/immudb/blob/v1.4.0/embedded/store/immustore.go#L1217
[immudb-txrelease]: https://github.com/codenotary/immudb/blob/v1.4.0/embedded/store/immustore.go#L1221
[immudb-mutex-pool]: https://github.com/codenotary/immudb/blob/v1.4.0/embedded/store/txpool.go#L39
[immudb-txalloc-error]: https://github.com/codenotary/immudb/blob/v1.4.0/embedded/store/immustore.go#L1047
[post-prev]: {{< relref "/content/post/2022-10-07-throttling-in-go.md" >}}

## Pool-throttler

Let's do some exercise now and create such a throttling pool.

I decided to use [generics] in the implementation. Instead of creating a pool that works on the `interface{}` type (or some other concrete one) we can leave the selection of that underlying type to the user of the pool. That way we can avoid a lot of conversions and type assertions ruling out unnecessary casting cost.

[generics]: https://go.dev/doc/tutorial/generics

As with tokens, the core of pool-based throttler will again be a channel:

```go
type poolLimiter[T any] struct {
    ch chan T
}
```

In case of token-based throttlers the initialization of such channel was simple because the token type was empty and did not contain any information. In case of a pool, objects in the pool must be initialized correctly. For that reason I decided to add a dedicated initialization functor when creating objects in the pool:

```go
func NewPoolLimiter[T any](poolSize int, gen func() T) PoolLimiter[T] {

    ret := &poolLimiter[T]{
        ch: make(chan T, poolSize),
    }

    // Preallocate the pool of objects
    for i := 0; i < poolSize; i++ {
        ret.ch <- gen()
    }

    return ret
}
```

And now the essence of the pool - acquiring and releasing resources. Let's write few implementations for different use cases, starting with the simplest one.

## Blocking acquire

The easiest acquire method to write is the blocking one that will wait until there is at least one object in the pool:

```go
func (p *poolLimiter[T]) Acquire() T {
    return <-p.ch
}
```

Nothing really fancy here - the channel gives us exactly the functionality that we need. But you may also have noticed that contrary to [the previous token-based implementations][golimiters-release] I decided not to return the release function from that method.

[golimiters-release]: https://github.com/byo/go-limiters/blob/v0.1.0/interfaces.go#L15

Instead there will be a dedicated `Release` function in the pool itself:

```go
func (p *poolLimiter[T]) Release(item T) {
    p.ch <- item
}
```

Why such change of the interface? I did some benchmarks and this way the implementation is almost twice as fast (41 ns/op vs 72 ns/op). The overhead of additional function value is pretty significant here. For that reason I also changed the interface of the original token-based throttler (that code even simplified which is usually a good sign).

As a side note, the code written so far looks too trivial, isn't it? Well, it just proves that golang's built-in primitives are really well designed.

## Non-blocking acquire

Let's move to something a bit more complex now - acquire that will return an error if the pool is empty:

```go
func (p *poolLimiter[T]) AcquireNoWait() (T, error) {
    var item T
    select {
    case item = <-p.ch:
        return item, nil
    default:
        return item, ErrResourceExhausted
    }
}
```

Nothing really fancy here - the built-in [select] is all we need.

The returned item is created and initialized at the beginning of this method which may be a bit surprising. This is needed if the pool is empty. In such case we still have to return some instance of the `T` type along with an error. Creating a default value initialized at the beginning does the trick.

[select]: https://go.dev/tour/concurrency/5

## Context-aware acquire

The most complex (but still fitting in just a few lines of code) is the context-aware acquire:

```go
func (p *poolLimiter[T]) AcquireCtx(ctx context.Context) (T, error) {
    var item T

    if ctx.Err() == nil {
        select {
        case item = <-p.ch:
            return item, nil
        case <-ctx.Done():
            break
        }
    }

    return item, ctx.Err()
}
```

The code is almost identical to the non-blocking acquire. If the pool is empty and we have to wait for some freed up resources, catching context cancellation with the [select] built-in allows the function to give up and return an error. Additional `if ctx.Err() == nil` ensures that we won't even try to fetch the resource if the context has already been cancelled.

## Exercise - token from pool

We can try to compare the newly written pool against the older token limiter. For that reason I've created a simple wrapper `struct` encapsulating the pool but exposing the token limiter interface instead:

```go
type poolLimiterToTokenLimiter struct {
    l PoolLimiter[struct{}]
}

func NewTokenLimiterFromPoolLimiter(maxTokens int) CancellableTokenLimiter {
    return &poolLimiterToTokenLimiter{
        l: NewPoolLimiter(maxTokens, func() struct{} { return struct{}{} }),
    }
}

func (l *poolLimiterToTokenLimiter) Acquire() {
    l.l.Acquire()
}

func (l *poolLimiterToTokenLimiter) AcquireNoWait() error {
    _, err := l.l.AcquireNoWait()
    return err
}

func (l *poolLimiterToTokenLimiter) AcquireCtx(ctx context.Context) error {
    _, err := l.l.AcquireCtx(ctx)
    return err
}

func (l *poolLimiterToTokenLimiter) Release() {
    l.l.Release(struct{}{})
}
```

Such implementation was roughly 30% slower than the dedicated token limiter implementation (48.54 ns/op vs 36.90 ns/op). My guess is that this is mostly caused by indirect calls and additional returned value. I think that the gap will become smaller in future versions of the go compiler that will be able to inline functions and detect dead code much better.

## Next

The topic of throttling is still not fully explored here so stay tuned for more updates.

And as before, you can find the full source code with all test and benchmarks in [Github repository][go_limiters].

[go_limiters]: https://github.com/byo/go-limiters/tree/v0.2.0
