# Rust 的异步状态

Rust 异步编程中部分功能支持，与同步编程一样具有相同的稳定性保障，
其它部分则仍在完善开发中，不断更新、改变。使用 Rust 异步，你将会：

- 在典型的并发工作负载中获得出色的运行性能。
- 更频繁地与高级语言功能交互，例如生命周期和固定（Pinning）。
- 同步和异步代码之间，及不同的异步实现之间的一些兼容性约束。
- 由于异步运行时及语言功能支持的不断发展，会面临更重的维护负担工作。

简尔言之，使用 Rust 异步编程相较同步编程，将更难以使用且带来更重的维护负担，
但它将为你带来一流的运行性能。Rust 异步编程的所有领域都在不断改进，
因此这些问题都将会随着时间更迭而消失。

## 语言和库的支持

虽然 Rust 自身支持异步编程，但大多数的异步应用程序都依赖于社区的 crate
所提供的功能。因此，你需要依赖于语言功能及库的混合支持：

- 标准库提供了最基本的特征、类型和函数，
  如 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) 
  特征。
- Rust 编译器原生支持 `async/await` 语法。
- [`futures`](https://docs.rs/futures/) crate 提供了众多实用的类型、宏和函数。
  它们可以使用在任何 Rust 异步程序中。
- 异步代码的执行、IO 和任务生功能成由“异步运行时”提供，例如 `Tokio` 和
  `async-std`。大多数异步程序和一些异步 crate 依赖于特定的运行时，详情请参阅
  [“异步生态系统”](../08_ecosystem/00_chapter_zh.md)。

你习惯于在 Rust 同步编程中使用的一些语言特性可能在 Rust 异步编程中尚不可使用。
需注意，Rust 不允许你在 traits 中声明异步函数，因此，你需要以变通的方式的实现它，
这可能会导致你的代码变得更加冗长。

## 编译和调试

在大部分情况下，Rust 异步编程中，编译器和运行时错误的工作方式与同步编程相同。
但有一些值得注意的区别：

### 编译错误

Rust 异步编译时使用同 Rust 同步编译中一样的严格要求标准来产生错误信息，
但由于 Rust 的异步通常依赖于更繁杂的语言特性，例如生命周期和固定（Pin）,
你可能会更频繁的遇到这些类型的错误。

### 运行时错误

每当编译器遇到异步函数时，它都会在后台生成一个状态机。
异步 Rust 中的堆栈追踪通常包含这些状态机的详细信息，以及来自运行时的函数调用。
因此，解读异步 Rust 的堆栈追踪可能比同步代码更复杂。

### 全新的故障模式

在 Rust 异步中，可能有一些新的故障模式，例如，你在异步上下文调用阻塞函数，
或者你没通正确地实现 `Future` 特征。编译器，甚至于有时单元测试也无法发现这些错误。
本书旨在，让你对这些基本概念都有着深刻的理解，从而避免踏入这些陷阱。

## 兼容性注意事项

异步代码和同步代码不能总是自由组合使用。例如，你不能直接从同步代码中调用异步函数。
同步和异步代码也倾向于使用不同的设计模式，这也使得编写用于不同环境的代码变得困难。

异步代码不能总是自由地组合。一些 crates 依赖于特定的异步进行时才能运行。
若如此，则一般会在 crate 的依赖列表中指定此依赖。

这些兼容问题会限制你的选择，因此请务必尽早调查、确定你需要哪些 crate 及异步运行时。
一旦你选定了某个运行时，就不会再担心兼容性的问题了。

## 性能特点

异步 Rust 的性能取决于你所使用的运行时的实现方式。
尽管为 Rust 提供异步运行时的库相对较新，
但它们在大多数实际工作负载中都表现的非常出色。

换句话说，大多数的异步生态系统都假定是一个多线程运行时。
而这使得它很难取得单线程异步程序理论上的性能优势，即更廉价的同步。
另一个被忽视的方面是对延迟敏感的任务，这对驱动、GUI 程序等非常重要。
这些任务需有操作系统的支持、选择适合的运行时，以便完成合理的调度。
在未来，会有更优秀的库来满足这些场景的需求。