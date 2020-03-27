- Feature Name: `run_to_completion_futures`
- Start Date: 2020-03-22
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC aims to bring a second `async` function type to Rust. Compared to todays
`async fn`s (introduced in [#2394](https://github.com/rust-lang/rfcs/pull/2394)),
the new async function type will always run to completion. It will not be possible
to forcefully abort the execution of such a function, while it's execution is
suspended in an `.await` point. In order to reach this goal, a new `Future` type
is introduced, which represents an async run-to-completion function.

# Motivation
[motivation]: #motivation

Rust added support for asynchronous functions in
[#2394](https://github.com/rust-lang/rfcs/pull/2394). `async fn`'s as of today
get translated by the compiler into a `Future` type, which was modelled similar
to `Future`s in the pre `async/await` era. `Future`s provide a `.poll()` method
which can be used to drive `Future`s to completion and to query a `Future`s
execution status. `Future`s can be dropped at any point in time to cancel an
ongoing async operation.

The latter ability made cancellation in async rust a lot easier to achieve than
in most other environments. However the design choice also introduced some
gaps, which the new async function type aims to fill.

## Support for completion based operations

Todays `Future`s do not allow us to support completion based operations in
a flexible and safe fashion. Completion based operations are operations which
follow the follow sequence:
1. we start an asynchronous request, and provide references to all necessary
  resources to the engine which executes the requests
2. we later receive a notification from the engine that the request finished
3. we acquire the result of the operation, which finalizes the operation.
  after this step all resources which had been borrowed for the duratino of the
  async operation can be reused.

**Example for such operations are:**
- IO operations provided through the IO completion port (IOCP) APIs on windows.
- IO operations provided through [`io-uring`](https://kernel.dk/io_uring.pdf)
  and [`io_submit`](https://manpages.debian.org/testing/manpages-dev/io_submit.2.en.html) 
  on Linux
- IO operations offered through `libusb`
- IO operations executed in Kernel space (e.g. disk command queues, etc)
- Operations that are composed on top of the described promitives

Async/await is theoretically powerful enough that it allows us to abstract over
such APIs with code like:

```rust
/// Transmit a single byte array using an IO completion based low-level API
async fn transmit_data() {
    let buffer: String = "Data to transfer".into();
    let data = buffer.as_bytes();
    let bytes_transferred = engine.send(data).await;
}
```

This is however not possible with todays `async fn`s. The reason for this is
that the `Future` produced by invoking the function can be dropped while the
execution is still in progress inside `engine.send(data).await`.
If this happens, the engine (which still would hold onto a pointer to the data)
would act on memory that does not represent the actual data to transmit anymore.
This would lead to undefined behavior.

One workaround to this problem is to use owned objects within completion based
operations. E.g.

```rust
async fn transmit_data() {
    let buffer: String = "Data to transfer".into();
    let data: Bytes = buffer.into();
    let (bytes_transferred, data) = engine.send(data).await;
}
```

The downside of this API is that it is a lot less flexible than the original
slice based API. We can now only call the API with a certain owned buffer type.
We also can't use cheap [async] stack based buffers anymore, and might need to
add support for atomic reference counting. Therefore this async abstraction
layer would not represent a zero-cost abstraction anymore, compared to the
original C version of those APIs.

By introducing run-to-completion async functions, we can model the APIs exactly
as described in the original example.

## Offer protection against accidental returns

For some function it is really important that it actually runs to completion.
The reason is typically that this function forms an atomic transaction. If the
transaction is cancelled in the middle, the objects that this transaction is
manipulating will end up in an invalid state.

With todays `async fn`, implementors of such a function do not have any guarantee
that their code runs to completion. The calling/polling side is outside of the view
of the function, and the caller might drop the produced `Future` early.

Implementors of such transaction can only protected themselves against drops of
`Future`s by using RAII guards, which allow to execute cleanup and/or rollback
code if the Future was not driven to completion. However since this cleanup code
is executed in a synchronous `drop()` method, it can not call any further async
code. This might however be necessary, e.g. if a file or stream must be flushed 
using async IO in order to ensure correctness. In order to work around this
problem implementors of some libraries block on a secondary futures executors
in their `drop` code, in order to be able to execute async cleanup code. This
can e.g. be seen [here](https://github.com/dignifiedquire/async-tar/blob/0739f53aaf805f493d3952adb7d1456b5ac6715e/src/builder.rs#L619-L622). However
this mechanism is neither performant nor necessarily safe. Blocking on an async
task in a destructor means all other async tasks in the root executor can now
no longer make progress for the duration of the `drop`. And if this mechanism
would be used inside a single-threaded executor a deadlock could occur,
since resources inside `block_on` can require the eventloop which drove the
original `async fn` to make progress - which won't happen since the thread is
blocked on waiting for the cleanup `Future`.

`async fn`s with run to completion semantics will naturally allow transactions
to run to completion - in the absence of panics. Cleanup code can just be regular
code at the end of a function, and no RAII guards are required. Therefore the
new function type will reduce complexity for such transactions.

In addition to that it will lower the chance of accidental errors. There does
not exist an implicit return path in `.await` anymore, which might have not been
handled properly.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The proposal adds 2 new items to Rust:
- A new `Future` type, which will reside in `core::task`. In the remains of this
  document we will refer to this type as `RunToCompletionFuture`.
  This is a working name, the actual name could be different and more concise.
- A new kind of asynchronous function which is guaranteed to run to completion.
  The type of function needs to be distinguishable from a regular async function.
  For the further explanation we will assume a modifer `completion` will be used
  to distinguish this function type from a regular `async fn`. However this is
  also a tool to present the concept. Any other kind of syntax could be used to
  disambiguate the types. Calling an `async completion fn` will produce a
  `RunToCompletionFuture`, whose associated `Output` type is the return type of
  the function.

In a similar fashion as the function

```rust
async fn hello_twice() -> IoResult {
    out.writeln("hello world").await;
    timer.delay(Duration::from_millis(2000)).await;
    out.writeln("hello world").await;
}
```

gets translated into 

```rust
fn hello_twice() -> Future<Output=IoResult>
```

our new run to completion function

```rust
async completion fn hello_twice_complete() -> IoResult {
    out.writeln("hello world").await;
    timer.delay(Duration::from_millis(2000)).await;
    out.writeln("hello world").await;
}
```

would get translated into 

```rust
fn hello_twice_complete() -> RunToCompletionFuture<Output=IoResult>
```

The difference between the two function is purely that `hello_twice_complete`
is always guaranteed to run to completion. It will not be possible to only
observe a single line of output - as long as the process does not get aborted.

## `await` operator usage

The `await` operator can be used inside an `async completion fn` to await
`Future`s as well as `RunToCompletionFuture`s. This means:
- `async completion fn`s will be able to `.await` `async completion fn`s,
  `async fn`s and call synchronous `fn`s:
  ```rust
  async completion fn f1() {}
  async fn f2() {}
  fn f3() {}

  async completion fn example() {
      f1().await; // This is OK
      f2().await; // This is also OK
      f3(); // And this also
  }
  ```
- However `async fn`s will only be able to `.await` `async fn`s and call
  synchronous `fn`s.
  The reasons for this is that an `async fn` could be dropped during its
  execution by its caller. It therefore does not fulfill the required guarantees
  to call a function which must run to completion - and which therefore can not
  be suspended without being resumed later on.
  ```rust
  async completion fn f1() {}
  async fn f2() {}
  fn f3() {}

  async fn example() {
      f1().await; // This is NOT allowed and should produce a compiler error
      f2().await; // This is OK
      f3(); // And this also
  }
  ```

## `async completion fn` support in runtimes

It is expected that runtimes (like [Tokio](https://tokio.rs/) or
[async-std](https://async.rs/)) would either add an additional `spawn` function
or modify their existing `spawn` function to accept a `RunToCompletionFuture`
instead of a `Future`. By adding this support, users can run all kinds of async
functions inside those executors. As long as normal `Future`s can be passed
to methods which accept a `RunToCompletionFuture`, this support can be added
in a backwards compatible fashion. It will not break code already running
on those runtimes.

## Cancellation support for run to completion functions

While an `async completion fn` can not be forcefully cancelled - by dropping the
`RunToCompletionFuture` it produced - they can still be cancelled in a cooperative
fashion. Cooperative cancellation consists of 3 phases:
1. Cancellation is signalled to the still running asynchronous function. This can
  e.g. performed by a `CancellationToken` object.
2. The Cancellation requests is detected within the asynchronous function. As a
  result of this cancellation request, the method **can** return to the caller -
  and still deliver a return value. It **could** however also continue to run for
  a certain time. E.g. in order to finalize the important transaction.
3. The issuer of the cancellation request waits for the cancelled async function
  to return. This can e.g. be achieved through a `WaitGroup` or `Semaphore` type.

The following example demonstrates cooperative cancellation with the use of a
`CancellationToken`:

```rust
async completion fn func_with_cancellation_support(&mut self, cancel_token: &CancellationToken) -> Option<i32> {
    let mut last_value = None;
    loop {
        select! {
            val = self.channel.receive() => {
                println!("Received value: {:?}", val);
                last_value = Some(val);
            },
            _ = cancel_token.cancel_notification().await => {
                // This method was cancelled. Return the last received result
                return last_value;
            }
        }
    }
}
```

`CancellationToken`s can directly be used in combination with the `select!`
macro to cancel operations/`Future`s which do not make use of
run-to-completion semantics. This would for example be async channels, timers,
mutexes, semaphores, etc.

In order to cancel run to completion operations, the `CancellationToken` must be 
forwarded up to a point where cancellation is checked. This would e.g.
require a library which e.g. utilizes `io_uring` to perform async IO operations
to take a `CancellationToken` as an argument and listen for the cancellation
request while the operation is still pending. The signature of such a method
would e.g. be

```rust
async completion fn read_with_uring(
    &mut self, buffer: &mut [u8], cancel_token: &CancellationToken
) -> Result<usize, IoError>;
```

The implementation of such a method would rather be complicated - but only needs
to occur within the runtime which provides the method. Most end user code is
simply expected to forward the `CancellationToken`:

```rust
async completion fn read_all_with_uring(
    reader: &mut Reader, buffer: &mut [u8], cancel_token: &CancellationToken
) -> Result<(), IoError> {
    let mut offset = 0;
    while offset != buffer.len() {
        let read = reader.read_with_uring(&buffer[offset..], cancel_token).await?;
        offset += read;
    }
    Ok(())
}
```

Runtimes could also decide to forward `CancellationToken`s implicitely through
task-local storage. Or the ability to forward a `CancellationToken` could be
added to the [`std::task::Context`](https://doc.rust-lang.org/1.42.0/std/task/struct.Context.html)
type in the future. Such a mechanism is however outside of the scope of this
proposal.

## Timeout handling with `async completion fn`

Another common case where a `select!` like functionality is needed besides
checking for cancellation is for timeouts. The current `select!` macro could not
be used to time-out on `async completion fn`, since `select!` does not drive
the aborted branches to completion.

**Therefore the following example is not valid**:

```rust
async completion fn read_with_timeout(
    reader: &mut Reader, buffer: &mut [u8], timeout: Duration
) -> Result<usize, IoError> {
    select! {
        read_result = reader.read_with_uring(&buffer[offset..]) => read_result
        _ = runtime::timer::delay_for(timeout) => Err(IoError::Timeout)
    }
}
```

Instead of this, a new mechanism is required which guarantees that the non
timeout branches will be cooperatively cancelled and driven to completion after
the timeout occured. That mechanism can be provided by runtimes in a variety of
fashions. They could either provide a variant of the `select!` macro which is
usable for `async completion fn`. Or they could provide a dedicated timeout
method:

```rust
async completion fn read_with_timeout(
    reader: &mut Reader, buffer: &mut [u8], timeout: Duration
) -> Result<usize, IoError> {
    runtime::with_timeout(timeout, async completion move {
        reader.read_with_uring(&buffer[offset..])
    }).await
}
```

In this example the necessary cancellation token is forwarded through a runtime
internal mechanism (e.g. task-local storage) from the timer to the read call.
However it could also be explicitly passed.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

As described earlier, an `async completion fn` would be transformed into a
`RunToCompletionFuture`.

## async transformation

The async transformation itself would happen mostly like the current `async fn` transformation. There is no need for difference, apart from generating a
different return type.

The compiler will mainly need to introduce an additional type check, which allows
`async completion fn`s to `await` other `async completion fn`s, while `async fn`s
can not await those.

## `RunToCompletionFuture` type definition

This RFC proposes to keep `RunToCompletionFuture` closely aligned to the current
`Future` type by defining it in the following fashion.

```rust
pub trait RunToCompletionFuture {
    /// The type of value produced on completion.
    type Output;

    /// Attempt to resolve the future to a final value, registering
    /// the current task for wakeup if the value is not yet available.
    ///
    /// # Return value
    ///
    /// This function returns:
    ///
    /// - [`Poll::Pending`] if the future is not ready yet
    /// - [`Poll::Ready(val)`] with the result `val` of this future if it
    ///   finished successfully.
    ///
    /// If a call to `poll()` call was issued to a `RunToCompletionFuture` that
    /// returned `Pending`, the caller **must** call `poll()` again later,
    /// until it returns `Ready`.
    ///
    /// Callers are not allowed `drop()` a future which returned `Pending` as
    /// its last poll result. Futures are only allowed to be dropped if they
    /// either had never been polled, or if the last `poll()` call returned `Ready`.
    unsafe fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

The type is thereby nearly identical to the existing `Future` trait. The only
difference is the added `unsafe` annotation. The trait is only safe to use if
the caller accepts `RunToCompletionFuture`s contract - which requires the caller
to poll it either never or exactly until it returns `Ready` once.

This contract guarantees that the function which is represented by this
`RunToCompletionFuture` gets driven to completion.
If an `async completion fn` would internally call another `async completion fn`
it would correspond to a `RunToCompletionFuture` which needs to poll another
`RunToCompletionFuture`. Since the poller of the outer `RunToCompletionFuture`
promises to drive it to completion, this `RunToCompletionFuture` can also promise
to the inner `RunToCompletionFuture` that it will drive it to completion.

While different designs of the trait might be possible - including ones which e.g.
use different types for starting the operation and driving the operation, staying
close the original `Future` trait has some benefits:
- We already have extensive experience with this trait, and know it works reasonably
  well for the intended use-cases.
- It will be easier for users to learn the two traits.
- The trait will stay object safe. We will be able to build traits on top if it -
  e.g. for IO completion based TCP streams - which still support dynamic dispatch.
- The task and waker system are completely unmodified, and will work the same
  fashion for both `Future` types.

# Drawbacks
[drawbacks]: #drawbacks

### Additional complexity through another `Future` and function type

The main drawback of adding another async function type is that it introduces
yet another concept that developers need to be aware of. It thereby increases
complexity of the language. Even if we add support for run to completion
`async fn`, developers still need to be aware about normal `async fn`, and the
fact that those behave different than normal functions and can return in
`await` points.

However even without adding this new feature, developers are expected to
experience the complexity of a variety of different behaviors in async functions.
They already need to be aware about the current cancellation behavior. They will
also need to be aware that cancelling some of the current async functions (e.g.
the proposed `scope` function for structured concurrency support in tokio) will
will lead to `panic`s or `abort`s, since there exists no way to handle a
cancellation in a reasonable fashion. By introducing run-to-completion function
we will at least gain a feature on type-system level that will help to disambiguate
functions which can safely be synchronously cancelled and ones which need to
run to completion.

### The new trait is `unsafe` by default

The new proposed trait requires the implementation of an `unsafe` method. The
`unsafe` annotation here is mainly required to enforce a special contract with
the caller which is currently not achievable within Rusts safe type system.

This is unfortunate, since we strive to minimize the use of `unsafe` code within
Rust codebases. The `unsafe` annotation also allows implementors of the trait
to use any arbitrary unsafe code, incl. code which could potentially lead to
memory unsafety issues. This is not necessarily intended, because we want
`unsafe` here mainly in order to enforce a contract with the caller.

However we do not expect a lot of people to implement `RunToCompletionFuture`
themselves. It should mainly be generated by the compiler. In addition this low
level primitives like IO completion based socket types would need to be implemented
by runtime authors - but those will need unsafe code anyway (they need to pass
raw pointers to underlying C code or the kernel for an amount of time which can
not be checked by lifetimes). Most users are purely expected to use high level
`async completion fn`s, which will not require unsafe code. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposal presented here provides a way to add run to completion semantics
for asynchronous functions in a backwards compatible and minimally invasive way.
Since neither the way an async function desugars into a generator/state-machine nor
the way wakeups (the task system) are performed are touched, the effort to integrate
this mechanism should be manageable.

The mechanism will allow us to provide true zero cost abstractions for completion
based operations. Those already exist in other languages (C++, Zig, C#, Kotlin,
etc), but not yet in Rust.

`async completion fn` will also introduce an async function type which behaves
more similar to normal functions than current `async fn`s do. It might therefore
have been interesting to see `async completion fn` as the new "default" async
function type. However this is not possible anymore, since `async fn` is already
stabilized.

And while the introduction of another type of async function type sounds like
as an additional complexity at first, it also has its benefits:

Users will continue to be able to write simple and safe `Future` implementations
with cancel-anytime semantics manually, and be able to use them in combination
with powerful flow-control macros like `select!` and `join!`. A lot of types
do not require run-to-completion semantics - e.g. async Channels, Mutexes and
Semaphores and Timers do very well without them. We might even prefer to cancel
those operations at any time. Only the operations which require run-to-completion
semantics can depend on them - and are still able to use other operations
internally. 

## Relation to `poll_drop`

A proposal which might be related is the one about
[`async drop / asynchronous destructors`](https://boats.gitlab.io/blog/post/poll-drop/). This blog post proposes to a `poll_drop` method to certain types,
which allows to run asynchronous code also in their cleanup phase.
However this mechanism would still not meet the motivations for this proposal.

The reason for this is that **`poll_drop` code is not guaranteed to be run**.
Since it is just an optional method on types, it might never be called.
This is especially likely to happen if `poll_drop` is just used deep within the
call hierarchy, while the code around it is not aware of it.

Any code which is simliar to [`select!`](https://docs.rs/futures/0.3.4/futures/macro.select.html), and which was not converted to be made aware of `poll_drop`
could cause `poll_drop` not be called. Therefore `poll_drop` is not a mechanism
which e.g. allows us to add a safe Rust APIs around async completion based
mechanism.

`async completion fn` in comparison enforces through the type system that
methods will run to completion, and any necessary cleanup code is run before 
resources are released.

Since `RunToCompletionFuture` is a new trait, it would be possible to add
`fn poll_drop(self: Pin<&mut Self>)` on this trait, and enforce users to of the
trait to use it: The `unsafe` contract in the trait could require callers to
either `.poll()` the future to completion, or to switch over to calling `poll_drop`.

However the benefit seems low. By doing this the Future would again move away
from one linear code path into an alternate one - which is what run to completion
wants to avoid. It is also highly likely that the code inside `poll_drop` would
actually be equal to the one `poll` itself, since `poll` is also expected to
perform any final cleanup.

A different alternative might be to pair run to completion methods later with
some kind of `finally` or `defer` blocks, which allow to run some (potentially
async) code also if methods are exiting early. The benefit of this approach is
that it could work for synchronous methods as well as for async methods.

# Prior art
[prior-art]: #prior-art

Examples for languages which added support for async/await are:
- C#
- Javascript
- C++ (Coroutines)
- Kotlin (Coroutines)
- Zig

In all these languages async functions have run to completion semantics. Rust
is the only language that is known to the author which allows to stop the
execution of an asynchronous function in the middle of the execution. By adopting
this RFC, Rust would gain a mechanism which optionally allows Rusts asynchronous
functions closer to the ones found in other languages, as well as closer to
synchronous functions.

The `CancellationToken/StopToken` approach, which provides the ability to
cooperatively cancel methods with run to completion semantics also had been
successfully deployed in these environments.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Syntax and naming questions

This RFC leaves the concrete syntax and naming for the new function and Future
type open. For the `Future` any name could be used.

For the new `async completion fn` syntax, a syntax needs to be chosen that
is allowed within the Rust 2018 rules. These might not allow for a new keyword
like `completion`.

## Sub-typing relationships between `Future` and `RunToCompletionFuture`

This RFC leaves it open whether there should be a type relation between
`Future` and `RunToCompletionFuture`, or whether we just need to teach the
compiler about both types, and when the `.await` of the other type is possible.

Normal `Future`s can be used whenever a `RunToCompletionFuture` is required.
In that case the `Future`s support cancellation by drop, but the caller would
not make use of the capabilty and always drive them to completion.

A blanket impl like the following might be a possibility::

```rust
impl<F, T> RunToCompletionFuture for F
where F: Future<Output=T> {
    type Output = T;

    unsafe fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // forward the poll call to the `Future` implementation
        <Self as Future>::poll(self, cx)
    }
}
```

# Future possibilities
[future-possibilities]: #future-possibilities

The RFC outlines a few additional ideas which will be interesting to look at
in the future.

This section reenumerates those:
- In order to forward `CancellationToken`s, which allow for cooperative
  cancellation in `async completion fn`s, the
  [`std::task::Context`](https://doc.rust-lang.org/1.42.0/std/task/struct.Context.html)
  type could be extended.
- A standardization of `CancellationToken` could also be taken into consideration,
  instead of leaving its definition purely to runtime implementations or other
  independent libraries.
- A standardization of IO traits for completion based IO objects (e.g. sockets
  and files backed by `io_uring`) could be taken into consideration. Those would
  would be variants of the proposed `AsyncRead/AsyncWrite` traits, which would
  require run to completion semantics and cooperative cancellation.
- `defer` or `finally` blocks could help to execute cleanup code in a unified
  fashion for synchronous and asynchronous code.
