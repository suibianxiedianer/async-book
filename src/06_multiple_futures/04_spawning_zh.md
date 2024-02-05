# `Spawning`

`Spawning` 使你可以在后台运行新的异步任务，这可以让我们在它运行时继续执行其它代码。

假设我们有一个 Web 服务器需要在接受、处理连接时而不阻塞主线程。为了实现这点，我们可以使用 `async_std::task::spawn`
函数来创建并运行一个新的任务来处理这个连接。`spawn` 函数接收一个 `feture` 并返回一个 `JoinHandle`，
它可用于等待任务完成后的结果。

```rust,edition2018
{{#include ../../examples/06_04_spawning/src/lib.rs:example}}
```

`spawn` 返回实现了 `Future` 特征的 `JoinHandle`，所以我们可以通过 `.await` 来获取此任务的结果。
但这将阻塞当前的任务，直至新生成的任务运行完成。如果不使用 `.await` 等待该任务，则程序将继续运行，
若此任务在当前函数运行结束前没有完成，则会取消此任务（即不等待结果，直接丢弃）。

```rust,edition2018
{{#include ../../examples/06_04_spawning/src/lib.rs:join_all}}
```

通常，我们使用 `async` 运行时提供的 `channels` 来进行主任务与派生任务的通信。
