# 特征中的 `async`

目前，`async fn` 不能在*稳定版 RUST*的 `Trait` 特性中使用。
自 2022 年 11 月 17日起，async-fn-in-trait 的 MVP 在编译器工具链的 nightly 版本上可以使用，
[详情请查看](https://blog.rust-lang.org/inside-rust/2022/11/17/async-fn-in-trait-nightly.html)。。。。

与此同时，若想在稳定版的特性中使用 `async fn` ，你也可以使用
[async-trait crate from crates.io](https://github.com/dtolnay/async-trait)。

注意，使用这些 trait 方法，在每次功能调用时都会导致堆内存分配。
这对大多数的程序来说，这样的成本代价是可接受的，但是，请仔细考虑，
是否在可能每秒产生上百万次调用的低端公共 API 上使用它。
