# Redirect stdin from inside rust application

I'm currently working on a parser crate written in Rust that is still to be
released publicly. Part of the remaining work is writing tests for the library
and as I was doing this I thought it would be interesting if I could test
reading directly from the standard input stream (`stdin`). Obviously the sane
and proper way would be to use an abstraction over streams (which I do have in
the library through the
[`Read`](https://doc.rust-lang.org/stable/std/io/trait.Read.html) trait) and
write the test against this (which I will do in the end), but then again I
couldn't drop the idea that `stdin` is already such an abstraction. So shouldn't
it be possible to mock `stdin` directly in the test?

In short; my goal was to somehow create a buffer in memory (i.e. not on disk)
and somehow "redirect" `stdin` to read input from this buffer.

## Fumbling around

The Rust standard library does currently not expose a way to redirect stdin,
but it turns out that Linux *does* have support for such an operation! The
usual way of doing this is through the c library function
[`freopen`](https://linux.die.net/man/3/freopen):

```c
FILE *freopen(const char *path, const char *mode, FILE *stream);
```

> The freopen() function opens the file whose name is the string pointed to by
> path and associates the stream pointed to by stream with it. 

We can access this function through the [`libc`
crate](https://docs.rs/libc/latest/libc/fn.freopen.html), however, the third
parameter is of type `FILE*`, commonly referred to as a "stream". For regular
files we can obtain a stream with the `fopen` function, but what about the
standard input stream like `stdin`? Unfortunately, the `libc` crate currently
does not expose [constants for stdin and its
friends](https://github.com/rust-lang/libc/issues/7). 

As the `libc` author notes, it is fairly easy to make a shim for this, and
there's already [a crate](https://crates.io/crates/libc-stdhandle) that does
this. But it involves compiling a C file in order to get these constants and I
wanted to avoid external crates for a simple task like this.

`libc` does however expose constants to the file descriptors such as
[`STDIN_FILENO`](https://docs.rs/libc/latest/libc/constant.STDIN_FILENO.html),
and we can use the POSIX `fdopen` function to obtain a stream from file descriptors!

But there are still other issues with `freopen`. The first parameter is a
string pointing to a path for a file, but I would like to have an in-memory
solution. But this is not really a hard requirement, a temporary file using
[`tmpfile`](https://linux.die.net/man/3/tmpfile) would probably suffice. The
bigger issue however is that `freopen` closes the existing stream (i.e.
`stdin`) meaning we can not recover the stream after the test is finished. This
is especially problematic for tests as Cargo links all tests together into a
single binary, meaning any test executed after will have an already closed
`stdin` stream.

## The solution

Since one of the issues was that `libc` did not expose `stdin` as a stream, but
it does have a constant for the file descriptor, I started to look for a
solution using file descriptors instead of stream. And since `freopen` is part of
the C library, it means it's built on top of Linux system calls somehow.

After studying the source code of `freopen`, I managed to locate what I think
is the core of the implementation, which turns out to boil down to [a single
system
call](https://github.com/bminor/glibc/blob/4e21c2075193e406a92c0d1cb091a7c804fda4d9/libio/freopen.c#L94-L96)!

```c
int dup3(int oldfd, int newfd, int flags);
```

> After a successful return from one of these system calls, the old and new
> file descriptors may be used interchangeably. 

This is what I needed! Furthermore, we can also use its sibling `dup` to get a
"copy" of `stdin` that we can safe for later to restore it.

Finally to put it to test! First, create the in-memory buffer:

```rust
let name = std::ffi::CString::new("input").unwrap();
let input = libc::memfd_create(name.as_ptr(), 0);
```

Even though the file is in-memory, we still need to give it a name. So we can
use the standard library's ffi types to make a compatible string.

Next, make a copy of `stdin`:

```rust
let stdin_copy = libc::dup(libc::STDIN_FILENO);
```

, and finally, for the "trick":

```rust
let result = libc::dup2(input, libc::STDIN_FILENO);
```

Here I just use the simpler `dup2` version which is behaves the same as `dup3`
but without the `flags` parameter.

To actually write to the buffer, we can use Rust's `File` type to make it
easier for us. We also need to rewind the file so that when we read from our
now redirected `stdin`, it's starting from the beginning of the data we wrote
to the buffer (the position of the buffer is otherwise at the end).

```rust
let mut input = unsafe { std::fs::File::from_raw_fd(input) };
input.write_all(b"hello ").unwrap();
input.rewind().unwrap();
```

Next, call the function we are testing. Here it will just read and write to `stdout`:

```rust
let mut buf = String::new();
std::io::stdin().read_line(&mut buf).unwrap();
println!("{buf}"); // Prints "hello" to stdout.
```

Finally we restore `stdin`:

```rust
let result = libc::dup2(stdin_copy, libc::STDIN_FILENO);
```

, and test that it works by reading from it:

```rust
  let mut buf = String::new();
  std::io::stdin().read_line(&mut buf).unwrap(); // Type e.g. "world".
  println!("{buf}"); // Prints "world" to stdout.
```

All put together into an executable, with some basic error handling, looks like
this:

```rust
use std::os::unix::io::FromRawFd;
use std::io::{Seek, Write};

fn main() {
    // create in-memory file
    let input = unsafe {
        // allocate name string suitable for c interface
        let name = std::ffi::CString::new("input").unwrap();
        let input = libc::memfd_create(name.as_ptr(), 0);
        if input == -1 {
            panic!();
        }
        input
    };

    // redirect stdin to in-memory file
    let stdin_copy = unsafe {
        // make copy of current stdin fd for recovering later
        let stdin_copy = libc::dup(libc::STDIN_FILENO);
        if stdin_copy == -1 {
            panic!();
        }
        // the magic trick! redirect our in-memory file to stdin
        let result = libc::dup2(input, libc::STDIN_FILENO);
        if result == -1 {
            panic!();
        }
        // store new fd for original stdin to recover
        stdin_copy
    };

    // create rust file for ease of use
    let mut input = unsafe { std::fs::File::from_raw_fd(input) };
    input.write_all(b"hello").unwrap();
    // rewind the file so we read from the beginning
    input.rewind().unwrap();

    // the "tested" code, just read from stdin and write to stdout
    let mut buf = String::new();
    std::io::stdin().read_line(&mut buf).unwrap();
    println!("{buf}");

    // restore stdin
    unsafe {
        let result = libc::dup2(stdin_copy, libc::STDIN_FILENO);
        if result == -1 {
            panic!();
        }
    }

    // and test that it works
    let mut buf = String::new();
    std::io::stdin().read_line(&mut buf).unwrap();
    println!("{buf}");
}
```

And if we execute this, and type "world" when prompted, we get output like:

```
hello
world
world
```

The first line is from our input read from the buffer, the second line from
what we type and third the same data echoed back!
