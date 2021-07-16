# The State of Asynchronous Rust
# Rust 的异步状态

Rust 异步编程中部分功能支持，与同步编程一样具有相同的稳定性保障，
其它部分则仍在按计划完善开发中，使用 Rust 异步，你将会：

- 在典型的并发工作负载中获得出色的运行性能。
- 更多的使用高级语言功能，例如生命周期和固定（Pinning）。
- 同步和异步代码之间，及不同的异步实现之间的一些兼容性约束。
- 由于异步运行时及语言功能支持的不断发展，更重的维护负担工作。

简尔言之，使用 Rust 异步编程相较同步编程，将更难以使用且带来更重的维护负担，
但它将为你带来一流的运行性能。Rust 异步编程的所有领域都在不断改进，
因此这些问题都将会随着时间更迭而消失。

## 语言和库的支持

虽然 Rust 本身支持异步编程，但大多数的异步应用程序都依赖于社区的 crate
所提供的功能。因此，你需要依赖于语言功能及库的混合支持：

- 标准库提供了最基本的特征、类型和函数，
  如 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) 
  特征。
- Rust 编译器原生支持 `async/await` 语法。
- [`futures`](https://docs.rs/futures/) crate 提供了众多实用的类型、宏和函数。
  它们可以使用在任何异步 Rust 程序中。
- 异步代码的执行、IO 和任务生功能成由“异步运行时”提供，例如 `Tokio` 和
  `async-std`。大多数异步程序和一些异步 crate 依赖于特定的运行时，详情请参阅
  [“异步生态系统”](../08_ecosystem/00_chapter.md)。

你习惯于在 Rust 同步编程中使用的一些语言功能在 Rust 异步编程中尚不可使用。
需注意，Rust 不允许你在 traits 中声明异步函数，因此，你需要以变通的方式的实现它，
这可能会导致你的代码变得更加冗长。

## 编译和调试

在大部分情况下，Rust 异步编程中，编译器和运行时错误的调试方法与同步编程相同。
但有一些值得注意的区别：

### 编译错误

Rust 异步编译时使用同 Rust 同步编译中一样的严格要求标准来产生错误信息，
但由于 Rust 的异步通常依赖于更繁杂的语言特性，例如生命周期和固定（Pin）,
你可能会更频繁的遇到这些类型的错误。

### 运行时错误

每当编译器遇到异步函数时，它都会在后台生成一个状态机。
异步 Rust 中的堆栈追踪通常包含这些状态机的详细信息，以及来自运行时的函数调用。
因此，它将比同步 Rust 产生更多的解释堆栈追踪信息。

### 全新的故障模式

A few novel failure modes are possible in async Rust, for instance
if you call a blocking function from an async context or if you implement
the `Future` trait incorrectly. Such errors can silently pass both the
compiler and sometimes even unit tests. Having a firm understanding
of the underlying concepts, which this book aims to give you, can help you
avoid these pitfalls.

## Compatibility considerations

Asynchronous and synchronous code cannot always be combined freely.
For instance, you can't directly call an async function from a sync function.
Sync and async code also tend to promote different design patterns, which can
make it difficult to compose code intended for the different environments.

Even async code cannot always be combined freely. Some crates depend on a
specific async runtime to function. If so, it is usually specified in the
crate's dependency list.

These compatibility issues can limit your options, so make sure to
research which async runtime and what crates you may need early.
Once you have settled in with a runtime, you won't have to worry
much about compatibility.

## Performance characteristics

The performance of async Rust depends on the implementation of the
async runtime you're using.
Even though the runtimes that power async Rust applications are relatively new,
they perform exceptionally well for most practical workloads.

That said, most of the async ecosystem assumes a _multi-threaded_ runtime.
This makes it difficult to enjoy the theoretical performance benefits
of single-threaded async applications, namely cheaper synchronization.
Another overlooked use-case is _latency sensitive tasks_, which are
important for drivers, GUI applications and so on. Such tasks depend
on runtime and/or OS support in order to be scheduled appropriately.
You can expect better library support for these use cases in the future.
