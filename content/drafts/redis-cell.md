---
hook: TODO
location: San Francisco
published_at: 2016-12-07T06:11:40Z
title: redis-cell
---

When I first noticed the Redis Labs module hackathon and started to brainstorm
project ideas, I settled on one quite quickly. I've been in industry long
enough to have seen Redis put to a huge variety of different uses--indeed it's
the Swiss Army knife of the modern production stack--but there's one place in
particular that I see it being brought to bear over and over again.

Web services that are exposed to a network tend to need various layers of
protection. The most common form of protection is of course authentication,
which ensures that the users accessing your resources are the ones that you
expect to be accessing them. Another very common one is controlling the _rate_
at which users allow to access those resources. This is obviously quite useful
for public services that need protection against (both intentionally and
unintentionally) malicious actors, but also for internal services to protect
against certain types of accidental use -- the thundering herd problem for
example, where many consumers wake up simultaneously and contend for access to
the same resource (like an API) simultaneously. Even companies like Google, who
are widely known for the excellence in technical competency, have admitted to
occasionally making this sort of mistake.

If you look at almost any commonly used APIs that you can find online, you'll
notice that the vast majority of them are controlling access using rate
limiters. GitHub, Spotify, Heroku, and Uber are all good examples.

A naive rate limiter implementation might simply track the number of operations
taken in the expected period of time and expire buckets as that period comes to
an end. Most real world rate limiters use a slightly more sophisticated
algorithm called "drip bucket". It's an easy metaphor that models rate limit
capacity as a bucket that has a fixed-size hole in its bottom. As a user
consumes operations, water is added to the bucket and its water level rises. If
the bucket becomes full, no more operations are allowed, but luckily, the hole
in the bucket is allowing water to escape at a constant rate. As long as the
rate of water in and the rate of water out stay roughly equal, the system stays
at equilibrium and operations are never limited.

Drip bucket is useful because its implementation is both computationally and
storage efficient, but also because it offers a good user experience by
providing a "rolling" time period. Even if a user accidentally burns through
their entire limit in a single burst, they'll have more limit available almost
immediately instead of having to wait for the next window to start.

redis-cell implements a variation of drip bucket called "generic cell rate
algorithm" (GCRA). It's funtionally identical, but uses some clever logic so
that each users being tracked needs only a single backend key to track their
entire state.

## Demonstration (#demonstration)

The module exposes a single command: `CL.THROTTLE`. It's invoked with
parameters that describe the rate limit that should be enforced. For example:

```
CL.THROTTLE user123 15 30 60 1
               ▲     ▲  ▲  ▲ ▲
               |     |  |  | └───── apply 1 operation (default if omitted)
               |     |  └──┴─────── 30 operations / 60 seconds
               |     └───────────── 15 max_burst
               └─────────────────── key "user123"
```

It responds with an array where the foremost element indicates whether the
request should be limited. The other elements contain quota and timing metadata
that's often returned from HTTP services as informative headers along with the
response. For example:

```
127.0.0.1:6379> CL.THROTTLE user123 15 30 60
1) (integer) 0   # 0 means allowed; 1 means denied
2) (integer) 16  # total quota (`X-RateLimit-Limit` header)
3) (integer) 15  # remaining quota (`X-RateLimit-Remaining`)
4) (integer) -1  # if denied, time until user should retry (`Retry-After`)
5) (integer) 2   # time until limit resets to maximum capacity (`X-RateLimit-Reset`)
```

Rolling your own rate limiting module is quite possible of course, but
redis-cell aims to provide a general and widely-useful implementation that can
be integrated into a project built with any programming language or framework,
just as long as its stack includes Redis.

## Rust (#rust)

redis-cell's other notable feature is the language that it's written in.
Although the Redis module API was originally intended to be consumed by another
C program (it's exposed as a C header file in `redismodule.h`), the project is
implemented in pure Rust. This is achieved through the use of the Rust FFI
(foreign function interface) module which allows the program to break out of
its normal safety rails and call directly into a systems level API. It's also
made possible because Rust programs are bootstrapped using only a tiny runtime,
and much like C programs, have no garbage collector.

So why bother? Well, although I could probably be considered to be nominally
literate in C, I don't have anywhere near the expertise to be confident that I
wouldn't write a program that contained a memory overrun or some other unsafe
operation that would manifest as a program-killing segmentation fault. As
evidenced by widespread issues like [Heartbleed][heartbleed], even highly
competent C developers are not beyond this class of mistake.

The rust compilers guarantees that all my memory accesses are safe, and its
strong type system goes a long way towards ensuring that I'm not accidentally
misusing code in a way that could cause a runtime problem. This is good for me,
but even better for would-be contributors the project; even someone who's never
written Rust before has only a miniscule chance of introducing a low-level
problem as long as they can get the program to compile.

### Example: Redis String Memory Safety (#redis-string-example)

Let's look at a simple example of this safety in action. The Redis module API
provides certain functions to allocate memory inside of Redis, and in the
default mode of operation, modules are tasked with freeing any memory that they
allocate in this way. So if `RedisModule_CreateString` is invoked to create a
new string, a call to `RedisModule_FreeString` is expected to eventually free
it.

In Rust, I wrap theses string with a higher level type so that I don't have to
work with them directly:

``` rust
pub struct RedisString {
    ctx: *mut raw::RedisModuleCtx,
    str_inner: *mut raw::RedisModuleString,
}
```

Now the trouble with manual memory management is that it can be dangerous. Say
I have a function that allocates a string, performs an operation with it, and
then frees the string before returning:

``` rust
fn run_operation() -> i64 {
    let s = RedisString::create(...);

    ...

    s.free();
    return 0;
}
```

Even if it works perfectly fine at first, it's easy for a bug to be introduced
somewhere down the line by someone not intimately familiar with the original
code. Say for example that a new return directive is introduced midway through
the function:

``` rust
fn run_operation() -> i64 {
    let s = RedisString::create(...);

    ...

    if error_occurred {
        // s is leaked!
        return 1;
    }

    ...

    s.free();
    return 0;
}
```

The new conditional branch doesn't free the string before leaving the function,
so the program now has a memory leak. This is a very easy mistake to make in C.

With Rust, we can do things a little differently. By implementing the
language's built-in `Drop` trait (think of a trait like an interface in most
languages), we can guarantee the memory safety of `RedisString`:

``` rust
impl Drop for RedisString {
    fn drop(&mut self) {
        self.free();
    }
}
```

`Drop` is like a destructor in C++; it's called when an instance of the type
goes out of scope. So in this case we ensure that our low-level free function
always gets called. There are no gotchas or failure conditions to worry about.

We don't even have to manually call `free` anymore. No matter how many new
conditional branches are introduced, memory safety is always guaranteed:

``` rust
fn run_operation() -> i64 {
    let s = RedisString::create(...);

    ...

    if error_occurred {
        // s is freed here automatically
        return 1;
    }

    ...

    // and here too!
    return 0;
}
```

## Acknowledgements (#acknowledgments)

Foremost I'd like to thank Redis Labs for hosting the Redis module hackathon
and giving me the opportunity to work on this. I also appreciated the help of
[Daniel Farina][dan] in detangling some of the Redis internals and figuring out
how its module system was laying out memory, and [Itamar Haber][itamar] for
giving me some instruction on the correct use of some of the Redis module APIs.

[dan]: https://twitter.com/danfarina
[heartbleed]: https://en.wikipedia.org/wiki/Heartbleed
[itamar]: https://twitter.com/itamarhaber
