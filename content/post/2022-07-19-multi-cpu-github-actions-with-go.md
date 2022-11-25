---
title: "Multi-CPU Github Actions with Go"
date: "2022-07-19"
tags: ["go", "howto", "github", "til", "ops", "qemu"]
---

When we think about CPU architecture usually there's one leader that comes to our mind - the famous [x86-64] one. It is the main player on our desktops and on the server side. Even more recent generation of game consoles switched to that architecture from some more exotic ones.

But there are alternatives. Some are pretty well known such as the [ARM] one that took over the mobile market (and slowly enters the desktop world with Apple M1 laptops). But there are many more like [RISC-V] or [SPARC], each with special properties and target group usually in the enterprise segment.

It is not a surprise that even newer languages such as [Go] support variety of CPU architectures. A quick check on my local Go compiler reveals that the for the Linux OS itself it already supports 12 CPU architectures:

```sh
$ go tool dist list | grep '^linux' | wc -l
12
```

[x86-64]: https://en.wikipedia.org/wiki/X86-64
[ARM]: https://en.wikipedia.org/wiki/ARM_architecture_family
[RISC-V]: https://en.wikipedia.org/wiki/RISC-V
[SPARC]: https://en.wikipedia.org/wiki/SPARC
[Go]: https://go.dev

## Automatic builds and tests with Github Actions

I can't imagine high quality project without CI pipeline checking if the code works as expected. With variety of tools and ready-to-use online services it's a must. Like Github actions. Setting a Github project with Github Actions is really simple. But if the target application should work on various CPU architectures, there's not much physical hardware easily available that we could choose from for our test workflows.

Github currently offers only x86-64 machines (with Windows, Linux or MacOS) which means that it will be hard to check if subtle CPU differences don't affect the behavior of our code. Those could be as simple as some assumption about the size of the `int` type, hardcoded endianness or different memory alignment.

## Qemu to the rescue

Qemu is a well-known and established software emulating various CPU architectures. And it turns out that it can easily be used to execute Linux binaries built for different CPU architectures.

Before we get into details, let me share a project where I demonstrate how it can be used: <https://github.com/byo/multi-cpu-demo/actions>. I created it to demonstrate application written in go that is compiled and tested on various CPU architectures. The essence is in [this workflow file][workflow]. Let's take a look at its most relevant sections.

[workflow]: <https://github.com/byo/multi-cpu-demo/blob/main/.github/workflows/test.yml>

### Setting up qemu

It turns out that there's already working Github action for that. Installing and setting up `qemu` for emulation is as simple as that:

```yaml
...
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      ...
      - uses: docker/setup-qemu-action@v2
      ...
```

That `setup-qemu-action` step itself does all the trickery to ensure the correct `qemu` version is installed and tightly integrated with the host (runner) operation system (more on that later).

### Compiling go code for target platform

Go compiler can easily generate binaries for various OSes and CPU architectures no matter what the host OS and architecture the compiler runs on.

To switch to different CPU we only have to setup the `GOARCH` environment variables:

```yaml
...
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      ...
      - run: go build -o hello .
        env:
          GOARCH: arm
      ...
```

Changing the target operating system can be done in a very similar way - by setting the `GOOS` environment variable. We won't be dealing with different OS-es though, only Linux this time :penguin::wink:.

### Executing binaries with emulation

This is something that I was really surprised with. The `qemu` instance set up with `docker/setup-qemu-action@v2` is actually installed in a very clever way. It does not simply install qemu binaries, actually I think it does not even copy a single binary to the host OS. Instead it is using [binfmt_misc] kernel mechanism to integrate `qemu` as an emulator needed to run certain binaries - in our case those are binary files for architecture other than the host system. If we try to run binary not meant for the host CPU, qemu takes over and does all the magic behind the scenes:

```yaml
...
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      ...
      - run: ./hello
      ...
```

[binfmt_misc]: <https://en.wikipedia.org/wiki/Binfmt_misc>

### Running go tests on different CPU architecture

The tight system integration of `qemu` has one more advantage. It allows running `go` tests just as if we were running those on the host CPU:

```yaml
...
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      ...
      - run: go test -v .
        env:
          GOARCH: arm
      ...
```

How does it work then? Does it mean that we're running the go toolchain itself in `qemu`?

Well, not really. When the `go test` command is executed, what it is actually doing is to compile a temporary binary from the test code and then executes it.

The `go` command itself is built for the host CPU which is `x86-64` in this case. It first runs on the host CPU and starts the test compilation process and here the `GOARCH` environment variable jumps in. The compiled temporary binary is built for the `GOARCH` target CPU architecture (`arm` in our example above).

When the `go` compiler tries then to execute that binary, `qemu` jumps in (through the tight integration with the host OS) and thus the test itself runs on CPU emulated with `qemu`. For the `go` command this is totally transparent.

## Trade-offs and gotchas

With the presented method we can run binaries on variety of different CPU architectures. But this does not use the OS emulation itself. The only thing we can change is the `GOARCH` variable and the `GOOS` must be left as `linux`. This can usually be worked around by using VM emulation or choosing different OS for the runner itself. There are cases like the [Apple M1][apple-m1] though where I didn't find so far a reliable way to run tests without buying a dedicated hardware box.

Also keep in mind that CPU emulation is a very heavy process itself. Binaries executed through emulation will run much slower than on the real hardware. There could also be subtle differences between the real and the emulated CPU.

It's also possible that the emulation itself will contain bugs. I experienced that myself before and even [posted a bogus bug report to the go compiler team][bug-report] only to figure out later that this was a bug in the qemu version that I was using.

Another fact worth mentioning here is that qemu does not emulate all CPU architectures supported by the go compiler (take a look at [this commit][disable-archs] from my example project). This means that some less popular configurations will still need either manual testing or dedicated hardware.

[apple-m1]: <https://en.wikipedia.org/wiki/Apple_M1>
[bug-report]: <https://github.com/golang/go/issues/53797>
[disable-archs]: <https://github.com/byo/multi-cpu-demo/commit/f02266ad6d3a26860876323c78ab0e27d98145e5>
