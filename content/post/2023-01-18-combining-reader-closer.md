---
title: "Quick way to combine io.Reader and io.Closer"
date: 2023-01-22
tags: ["go", "til", "code"]
---

There are many interesting tools in Golang's standard library to wrap [`io.Reader`][io.Reader] instance such as [`io.LimitedReader`][io.LimitedReader] or [`cipher.StreamReader`][cipher.StreamReader]. But when wrapping a [`io.ReadCloser`][io.ReadCloser] instance, the `Close` method is hidden.

Here's a quick code snippet to combine wrapped [`io.Reader`][io.Reader] and the original [`io.Closer`][io.Closer] through an inline `struct` to rebuild the [`io.Closer`][io.Closer] interface.

<!--more-->

```go
var rc io.ReadCloser = struct {
    io.Reader
    io.Closer
}{
    Reader: r,
    Closer: c,
}
```

## What it is about?

The [`io.Reader`][io.Reader] interface in Golang is a very powerful abstraction when streaming data. It can be used to read files, http responses, even raw byte arrays using a generic code:

```go
func sumAllBytes(r io.Reader) (uint64, error) {
    var buff [512]byte
    var sum uint64

    for {
        n, err := r.Read(buff[:])
        for i := 0; i < n; i++ {
            sum += uint64(buff[i])
        }

        if errors.Is(err, io.EOF) {
            return sum, nil
        }
        if err != nil {
            return sum, err
        }
    }
}
```

That method can be used to process files, http responses, even memory buffers:

```go
func main() {

    // Process a buffer
    buff := bytes.NewReader([]byte{0x01, 0x02, 0x03, 0x04})

    sum, err := sumAllBytes(buff)
    if err != nil {
        log.Fatal(err)
    }
    log.Println("Sum from buffer:", sum)

    // Process a http response

    resp, err := http.Get("https://www.google.com")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    sum, err = sumAllBytes(resp.Body)
    if err != nil {
        log.Fatal(err)
    }
    log.Println("Sum from http:", sum)

    // Process a file

    fl, err := os.Open("/some/file/path")
    if err != nil {
        log.Fatal(err)
    }
    defer fl.Close()

    sum, err = sumAllBytes(fl)
    if err != nil {
        log.Fatal(err)
    }
    log.Println("Sum from file:", sum)
}
```

## Cleaning up

In many cases (like the http and file instances above), the reader has to be explicitly closed to avoid resource leaks. For that reason, many resources are using the [`io.ReadCloser`][io.ReadCloser] interface and cleaning up can easily be achieved with a `defer rc.Close()` statement. But there are some cases where the close method is not called at the same function but somewhere at the caller site. At that construct, the caller is responsible for cleanup:

```go
func getDataStream(name string) (io.ReadCloser, error) {
    switch name {
    case "file":
        return os.Open("/some/file")
    case "http":
        resp, err := http.Get("https://www.google.com/")
        if err != nil {
            return nil, err
        }
        return resp.Body, nil
    case "buffer":
        return io.NopCloser(
            bytes.NewReader([]byte{0x01, 0x02, 0x03, 0x04}),
        ), nil
    default:
        return nil, errors.New("Invalid data stream name")
    }
}

func printSum(streamName string) {
    stream, err := getDataStream(streamName)
    if err != nil {
        log.Fatal(err)
    }
    defer stream.Close()

    sum, err := sumAllBytes(stream)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Sum of bytes in stream '%s' is '%d\n", streamName, sum)
}
```

So far so good, nothing to worry about. But let's extend this example with some stream wrapping:

```go
func getDataStream(name string) (io.ReadCloser, error) {
    switch name {
    case name == "file":
        return os.Open("/some/file")
    case name == "http":
        resp, err := http.Get("https://www.google.com/")
        if err != nil {
            return nil, err
        }
        return resp.Body, nil
    case name == "buffer":
        return io.NopCloser(
            bytes.NewReader([]byte{0x01, 0x02, 0x03, 0x04}),
        ), nil

    // v---  Create a truncated stream by applying limit over the base one ---v
    case strings.HasPrefix(name, "limit:"):
        r, err := getDataStream(name[6:])
        if err != nil {
            return nil, err
        }
        return io.LimitReader(r, 3)

    default:
        return nil, errors.New("Invalid data stream name")
    }
}
```

Unfortunately this code does not compile and ends up with this error:

```plain
cannot use io.LimitReader(r, 100) (value of type io.Reader) as type io.ReadCloser in return statement:
    io.Reader does not implement io.ReadCloser (missing Close method)
```

Wrapping the [`io.ReadCloser`][io.ReadCloser] with [`io.LimitedReader`][io.LimitedReader] does hide the [`io.Closer`][io.Closer] functionality of the original instance. And it turns out that there are many places in golang standard lib where such wrapping takes place.

## Inline struct to the rescue

There's an easy trick to bring back the `Close` method from the original reader back to the wrapped one:

```go
func getDataStream(name string) (io.ReadCloser, error) {
    switch name {
    case name == "file":
        return os.Open("/some/file")
    case name == "http":
        resp, err := http.Get("https://www.google.com/")
        if err != nil {
            return nil, err
        }
        return resp.Body, nil
    case name == "buffer":
        return io.NopCloser(
            bytes.NewReader([]byte{0x01, 0x02, 0x03, 0x04}),
        ), nil

    // v---  Create a truncated stream by applying limit over the base one ---v
    case strings.HasPrefix(name, "limit:"):
        r, err := getDataStream(name[6:])
        if err != nil {
            return nil, err
        }

        limitReader := io.LimitReader(r, 3)

        return struct {
            io.Reader
            io.Closer
        }{
            Reader: limitReader, // Read method will come from the wrapped reader
            Closer: r,           // Close method will come from the original reader
        }, nil

    default:
        return nil, errors.New("Invalid data stream name")
    }
}
```

## How does it work?

The inline `struct` contains two [embedded fields][embedded_fields], one for the reader and the other for the closer.

Since those fields are anonymous, the struct itself *inherits* methods from those fields as if those were declared on the struct. By doing so, whenever the compiler tries to cast the struct to some interface, it can *promote* those methods to fulfil the requirements of the interface.

In the code above we return an instance of [`io.ReadCloser`][io.ReadCloser] interface that requires both `Read` and `Close` method - and those are *borrowed* from embedded fields respectively.

Interestingly, if we would use whole [`io.ReadCloser`][io.ReadCloser] as the second embedded field instead of [`io.Reader`][io.Reader], the compiler (go 1.19 as of writing) throws an error which is caused by ambiguity between promoted field members (the `Read` method is not promoted due to ambiguity):

```plain
cannot use struct{io.Reader; io.ReadCloser}{…} (value of type struct{io.Reader; io.ReadCloser}) as type io.ReadCloser in return statement:
    struct{io.Reader; io.ReadCloser} does not implement io.ReadCloser (missing Read method)
```

[io.Reader]: https://pkg.go.dev/io#Reader
[io.Closer]: https://pkg.go.dev/io#Closer
[io.ReadCloser]: https://pkg.go.dev/io#ReadCloser
[io.LimitedReader]: https://pkg.go.dev/io#LimitedReader
[cipher.StreamReader]: https://pkg.go.dev/crypto/cipher#StreamReader
[embedded_fields]: https://go.dev/ref/spec#EmbeddedField
