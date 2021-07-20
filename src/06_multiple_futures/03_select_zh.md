# `select!`

`futures::select` 宏会同时运行多个 futures，当其中任何一个 future
完成后，它会立即给用户返回一个响应。

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:example}}
```

上面的函数将同时运行 `t1` 和 `t2`。当其中任意一个任务完成后，
就会运行与之对应的 `println!` 语句，同时结束此函数，无论是否还有未完成任务。

`select` 的基本语法是这样 `<pattern> = <expression> => <code>,`，
像这样你可以在 select 代码块里放进所有你需要的 futures。

## `default => ...` and `complete => ...`

`select` 同样支持 `default` 和 `complete` 分支。

当 `select` 中的 futures 都是未完成状态时，将运行 `default` 分支。
因此具有 `default` 分支的 `select` 都将立即返回一个结果。

在 `select` 的所有分支都是已完成状态，不会再取得任何进展时，`complete`
分支将会运行。当在循环中使用 `select!` 时，这是非常有用的！

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:default_and_complete}}
```

## 与 `Unpin` 和 `FusedFuture` 交互

One thing you may have noticed in the first example above is that we
had to call `.fuse()` on the futures returned by the two `async fn`s,
as well as pinning them with `pin_mut`. Both of these calls are necessary
because the futures used in `select` must implement both the `Unpin`
trait and the `FusedFuture` trait.

`Unpin` is necessary because the futures used by `select` are not
taken by value, but by mutable reference. By not taking ownership
of the future, uncompleted futures can be used again after the
call to `select`.

Similarly, the `FusedFuture` trait is required because `select` must
not poll a future after it has completed. `FusedFuture` is implemented
by futures which track whether or not they have completed. This makes
it possible to use `select` in a loop, only polling the futures which
still have yet to complete. This can be seen in the example above,
where `a_fut` or `b_fut` will have completed the second time through
the loop. Because the future returned by `future::ready` implements
`FusedFuture`, it's able to tell `select` not to poll it again.

Note that streams have a corresponding `FusedStream` trait. Streams
which implement this trait or have been wrapped using `.fuse()`
will yield `FusedFuture` futures from their
`.next()` / `.try_next()` combinators.

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:fused_stream}}
```

## Concurrent tasks in a `select` loop with `Fuse` and `FuturesUnordered`

One somewhat hard-to-discover but handy function is `Fuse::terminated()`,
which allows constructing an empty future which is already terminated,
and can later be filled in with a future that needs to be run.

This can be handy when there's a task that needs to be run during a `select`
loop but which is created inside the `select` loop itself.

Note the use of the `.select_next_some()` function. This can be
used with `select` to only run the branch for `Some(_)` values
returned from the stream, ignoring `None`s.

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:fuse_terminated}}
```

When many copies of the same future need to be run simultaneously,
use the `FuturesUnordered` type. The following example is similar
to the one above, but will run each copy of `run_on_new_num_fut`
to completion, rather than aborting them when a new one is created.
It will also print out a value returned by `run_on_new_num_fut`.

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:futures_unordered}}
```
