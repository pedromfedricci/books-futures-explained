# Conclusion and exercises

Congratulations. Good job! If you got this far you must have stayed with me
all the way. I hope you enjoyed the ride!

Remember that you call always leave feedback, suggest improvements or ask questions
in the [issue_tracker](https://github.com/cfsamson/books-futures-explained/issues) for this book.
I'll try my best to respond to each one of them.

I'll leave you with some suggestions for exercises if you want to explore a little further below.

Until next time!

## Further notes and comments

In [Chapter 2](1_futures_in_rust.md), I mentioned that it is common for the
executor to create a new Waker for each Future that is registered with the
executor, but that the Waker is a shared object similar to a `Arc<T>`.  One of
the reasons for this design is that it allows different Reactors the ability to
Wake a Future.  As an example of how this can be used, consider how you could
create a new type of Future that has the ability to be canceled.  A method of
doing this would be to add an
[`AtomicBool`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html)
to the instance of the future, and an extra method called `cancel()`.  The
`cancel()` method  will first set the
[`AtomicBool`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html)
to signal that the future is now canceled, and then immediately call instance's
own  copy of the Waker.  Once the executor starts executing the Future, the
_Future_ will know that it was canceled, and will do the appropriate cleanup
actions to terminate itself.

The main reason for designing the Future in this manner is because we don't have
to modify either the Executor or the other Reactors; they are all oblivious to
the change.  The only possible issue is with the design of the Future itself; a
Future that is canceled still needs to terminate correctly according to the
rules outlined in the docs for
[`Future`](https://doc.rust-lang.org/std/future/trait.Future.html).  That means
that it can't just delete it's resources and then sit there; it needs to return
a value.  It is up to you to decide if a canceled future will return
[`Pending`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Pending)
forever, or if it will return a value in
[`Ready`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready). Just
be aware that if other Futures are `await`ing it, they won't be able to start
until [`Ready`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready)
is returned.  A common technique for cancelable Futures is to have them return a
Result with an error that signals the Future was canceled; that will permit any
Futures that are awaiting the canceled Future a chance to progress, with the
knowledge that the Future they depended on was canceled.  There are additional
concerns as well, but beyond the scope of this book.  Read the documentation and
code for the [`futures`](https://crates.io/crates/futures) crate for a better
understanding of what the concerns are.

## Reader exercises

So our implementation has taken some obvious shortcuts and could use some improvement.
Actually digging into the code and try things yourself is a good way to learn. Here are
some good exercises if you want to explore more:

### Avoid wrapping the whole `Reactor` in a mutex and pass it around

First of all, protecting the whole `Reactor` and passing it around is overkill. We're only
interested in synchronizing some parts of the information it contains. Try to refactor that
out and only synchronize access to what's really needed.

I'd encourage you to have a look at how async_std solves this with a global [runtime](https://github.com/async-rs/async-std/blob/b446cd023084a157b8a531cff65b8df37750be58/src/rt/mod.rs#L15-L23) which includes the [reactor](https://github.com/async-rs/async-std/blob/b446cd023084a157b8a531cff65b8df37750be58/src/rt/runtime.rs#L39-L70)
and [how tokio's rutime](https://github.com/tokio-rs/tokio/blob/19a87e090ed528001e0363a30f6165304a710d49/tokio/src/runtime/context.rs#L2-L12) solves the same
thing in a slightly different way to get some inspiration.

* Do you want to pass around a reference to this information using an `Arc`?
* Do you want to make a global `Reactor` so it can be accessed from anywhere?

### Building a better executor

Right now, we can only run one and one future only. Most runtimes has a `spawn`
function which let's you start off a future and `await` it later so you
can run multiple futures concurrently.

As I suggested in the start of this book, visiting [@stjepan'sblog series about implementing your own executors](https://web.archive.org/web/20200207092849/https://stjepang.github.io/2020/01/31/build-your-own-executor.html)
is the place I would start and take it from there.

### Create an unique Id for each task

As we discussed in the end of the main example. What happens if the user pass in
the same Id for both events?

```rust, ignore
let future1 = Task::new(reactor.clone(), 1, 1);
let future2 = Task::new(reactor.clone(), 2, 1);
```

Right now it will deadlock since our `poll` method thinks the first poll of
`future2` is `future1` getting polled again and swaps out the waker with the
one from `future2`. This waker never gets called since the task is never
registered.

It's probably a bad idea to expose the user to this behavior, so we
should have a unique Id for each task which we use internally. There are a
many ways to solve this. Below is two suggestions to get going:

1. Let the reactor have a `usize` which is incremented on every task creation
2. Use a crate like `Guid` to generate an unique Id for each task

## Further reading

There are many great resources. In addition to the RFCs and articles I've already
linked to in the book, here are some of my suggestions:

[The official Asyc book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)

[The async_std book](https://book.async.rs/)

[Aron Turon: Designing futures for Rust](https://aturon.github.io/blog/2016/09/07/futures-design/)

[Steve Klabnik's presentation: Rust's journey to Async/Await](https://www.infoq.com/presentations/rust-2019/)

[The Tokio Blog](https://tokio.rs/blog/2019-10-scheduler/)

[Stjepan's blog with a series where he implements an Executor](https://stjepang.github.io/)

[Jon Gjengset's video on The Why, What and How of Pinning in Rust](https://youtu.be/DkMwYxfSYNQ)

[Withoutboats blog series about async/await](https://boats.gitlab.io/blog/post/2018-01-25-async-i-self-referential-structs/)

[condvar_std]: https://doc.rust-lang.org/stable/std/sync/struct.Condvar.html
[condvar_wiki]: https://en.wikipedia.org/wiki/Monitor_(synchronization)#Condition_variables
[arcwake]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.13/futures/task/trait.ArcWake.html