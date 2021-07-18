# 应用：构建一个执行器

Rust 的 `Future` 是懒惰的：除非积极地推动它完成，不然它不会做任何事情。
一种推动 future 完成的方式是在 `async` 函数中使用 `.await`，
但这只是将问题推进了一层，还面临着：谁将运行从顶级 `async` 返回的 future？
很明显我们需要一个 `Future` 执行器。

`Future` 执行器获取一级顶级 `Future`s 并在 `Future`
取得工作进展时通过调用 `poll` 来将它们运行直至完成。
通常，执行器会调用一次 `poll` 来使 future 开始运行。
当 `Future` 通过调用 `wake()` 表示它们已就绪时，会被再次放入队列中以便 `poll`
再次调用，重复直到 `Future` 完成。

在本章中，我们将编写一个简单的，能够同时运行大量顶级 futures 的执行器。

在这个例子中，我们依赖于 `futures` 箱，它提供了 `ArcWake` 特征，
有了这个特征，我们可以很方便的构建一个 `Waker`。

```toml
[package]
name = "xyz"
version = "0.1.0"
authors = ["XYZ Author"]
edition = "2018"

[dependencies]
futures = "0.3"
```

接下来，我们需要在 `src/main.rs` 的顶部导入以下路径：

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:imports}}
```

Our executor will work by sending tasks to run over a channel. The executor
will pull events off of the channel and run them. When a task is ready to
do more work (is awoken), it can schedule itself to be polled again by
putting itself back onto the channel.

In this design, the executor itself just needs the receiving end of the task
channel. The user will get a sending end so that they can spawn new futures.
Tasks themselves are just futures that can reschedule themselves, so we'll
store them as a future paired with a sender that the task can use to requeue
itself.

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:executor_decl}}
```

Let's also add a method to spawner to make it easy to spawn new futures.
This method will take a future type, box it, and create a new `Arc<Task>` with
it inside which can be enqueued onto the executor.

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:spawn_fn}}
```

To poll futures, we'll need to create a `Waker`.
As discussed in the [task wakeups section], `Waker`s are responsible
for scheduling a task to be polled again once `wake` is called. Remember that
`Waker`s tell the executor exactly which task has become ready, allowing
them to poll just the futures that are ready to make progress. The easiest way
to create a new `Waker` is by implementing the `ArcWake` trait and then using
the `waker_ref` or `.into_waker()` functions to turn an `Arc<impl ArcWake>`
into a `Waker`. Let's implement `ArcWake` for our tasks to allow them to be
turned into `Waker`s and awoken:

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:arcwake_for_task}}
```

When a `Waker` is created from an `Arc<Task>`, calling `wake()` on it will
cause a copy of the `Arc` to be sent onto the task channel. Our executor then
needs to pick up the task and poll it. Let's implement that:

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:executor_run}}
```

Congratulations! We now have a working futures executor. We can even use it
to run `async/.await` code and custom futures, such as the `TimerFuture` we
wrote earlier:

```rust,edition2018,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:main}}
```

[task wakeups section]: ./03_wakeups.md
