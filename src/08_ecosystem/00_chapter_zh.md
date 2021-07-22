# 异步的生态系统
Rust 目前只提供了编写异步代码的基础必要的功能。重要的是，标准库尚未提供像
执行器（executors)、任务（tasks)、反应器（reactors)、组合器（combinators）
与低级 I/O 的功能和特征。目前，是由社区提供的异步生态来填补这些空白的。

Rust 异步创建团队，希望在 Async Book 中扩展更多的示例来涵盖多种运行时。
如果你有兴趣为这个项目做贡献，可以通过
[Zulip](https://rust-lang.zulipchat.com/#narrow/stream/201246-wg-async-foundations.2Fbook)
联系我们。

## 异步 运行时
异步运行时，是用来执行异步程序的库。运行时通常和一个反应器及一个或多个执行器组成。
反应器为外部事务提供订阅机制，如异步 I/O、进程间通信和计时器。
在一个异步运行时里，订阅者通常是代表低级 I/O 操作的 futures。
执行器则负责任务的调试和运行。
它们会保持追踪在运行或中断中的任务，当任务可取得进展（就绪）时唤醒它们，
促使（通过 poll）futures 完成。
“执行器”和“运行时”这两个词，通常我们可能互换使用。
这里，我们使用“生态系统”这个词来形容一个绑定了兼容特征及具其特性的运行时。

## 社区提供的异步箱（Crates）

### The Futures Crate
[`futures` crate](https://docs.rs/futures/)
包括了在编写异步代码时实用的特征和方法。
它提供了 `Stream`、`Sink`、`AsyncRead` 和 `AsyncWrite` 特征，
及实用程序，如组合器。而这些实用程序和特征，也许最终会添加到标准库里！

`futures` 有自己的执行器，但没自己的反应器，所以它不支持计时器 futures
和异步 I/O 的执行。
所以，它不是一个完整的运行时。
通常我们会选择 `futures` 中的实用工作，并配合其它 crate 中的执行器来使用。

### 受欢迎的异步运行时
因为标准库中不提供异步运行时，所以并没有所谓的官方推荐。
下面的这些 crates 提供了一些受大家喜爱的运行时。
- [Tokio](https://docs.rs/tokio/)：一个受欢迎的异步生态系统，提供了 HTTP，gRPC
及跟踪框架。
- [async-std](https://docs.rs/async-std/)：一个为标准库提供异步功能组件的 crate。
- [smol](https://docs.rs/smol/)：一个小且简洁实用的异步运行时。
提供了 `Async` 特征，可用来装饰结构体，如 `UnixStream` 或 `TcpListener`。
- [fuchsia-async](https://fuchsia.googlesource.com/fuchsia/+/master/src/lib/fuchsia-async/)：
Fuchsia OS 中使用的执行器。

## 确定生态系统兼容性
并非所有的异步程序、框架和库都互相兼容，不同的操作系统和架构也是如此。
大部分的异步代码都可在任何生态系统中运行，但一些框架和库只能在特定的生态上使用。
生态系统的限制性不总被提及、记录破案，但有几个经典法则可帮助我们，
来确定库、特征和方法是否依赖于特定的生态系统。

任何包括异步 I/O、计时器、跨进程通信或产生任务的异步代码，
都依赖于一个特殊的执行器或反应器。
而其它异步代码，如异步表达式、组合器、同步类型和流，一般是独立于生态系统的，
前提是其内嵌套的 futures 也得是独立于生态系统存在的。
在开始一个项目之前，建议首先对使用到的异步框架、库做一个调查，
来确保你选择的运行时对它们有着良好的兼容性。

请注意，`Tokio` 使用 `mio` 作为反应器，且定义了自己的异步 I/O 特征，
包括 `AsyncRead` 和 `AsyncWrite`。就其本身而言，它不兼容依赖于
[`async-executor` crate](https://docs.rs/async-executor) 的`async-std` 和
`smol`，及定义了 `AsyncRead`、`AsyncWrite` 特征的 futures。

Conflicting runtime requirements can sometimes be resolved by compatibility layers
that allow you to call code written for one runtime within another.
For example, the [`async_compat` crate](https://docs.rs/async_compat) provides a compatibility layer between
`Tokio` and other runtimes.

Libraries exposing async APIs should not depend on a specific executor or reactor,
unless they need to spawn tasks or define their own async I/O or timer futures.
Ideally, only binaries should be responsible for scheduling and running tasks.

## Single Threaded vs Multi-Threaded Executors
Async executors can be single-threaded or multi-threaded.
For example, the `async-executor` crate has both a single-threaded `LocalExecutor` and a multi-threaded `Executor`.

A multi-threaded executor makes progress on several tasks simultaneously.
It can speed up the execution greatly for workloads with many tasks,
but synchronizing data between tasks is usually more expensive.
It is recommended to measure performance for your application
when you are choosing between a single- and a multi-threaded runtime.

Tasks can either be run on the thread that created them or on a separate thread.
Async runtimes often provide functionality for spawning tasks onto separate threads.
Even if tasks are executed on separate threads, they should still be non-blocking.
In order to schedule tasks on a multi-threaded executor, they must also be `Send`.
Some runtimes provide functions for spawning non-`Send` tasks,
which ensures every task is executed on the thread that spawned it.
They may also provide functions for spawning blocking tasks onto dedicated threads,
which is useful for running blocking synchronous code from other libraries.
