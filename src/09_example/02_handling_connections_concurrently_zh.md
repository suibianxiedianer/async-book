# 并发地处理连接
目前我们的代码中的问题是 `listener.incoming()` 是一个阻塞的迭代器。
执行器无法在 `listener` 等待一个入站连接时，运行其它 futures，
这便导致了我们只有等之前的请求完成，才能处理新的连接。

我们可以通过将 `listener.incoming()` 从一个阻塞迭代器转变为一个非阻塞流。
流类似于迭代器，但它是可用于异步消费的，
详情可回看[Streams](../05_streams/01_chapter_zh.md)。

让我们使用 `async-std::net::TcpListener` 替代 `std::net::TcpListener`，
并更新我们的连接处理函数，让它接受 `async_std::net::TcpStream`：
```rust,ignore
{{#include ../../examples/09_04_concurrent_tcp_server/src/main.rs:handle_connection}}
```

这个异步版本的 `TcpListener` 为 `listener.incoming()` 实现了 `Stream` 特征，
这带来了两个好处。其一，`listener.incoming()` 不再是一个阻塞的执行器了。
在没有入站的 TCP 连接可取得进展时，它可以允许其它挂起的 futures 去执行。

The second benefit is that elements from the Stream can optionally be processed concurrently,
using a Stream's `for_each_concurrent` method.
Here, we'll take advantage of this method to handle each incoming request concurrently.
We'll need to import the `Stream` trait from the `futures` crate, so our Cargo.toml now looks like this:
```diff
+[dependencies]
+futures = "0.3"

 [dependencies.async-std]
 version = "1.6"
 features = ["attributes"]
```

Now, we can handle each connection concurrently by passing `handle_connection` in through a closure function.
The closure function takes ownership of each `TcpStream`, and is run as soon as a new `TcpStream` becomes available.
As long as `handle_connection` does not block, a slow request will no longer prevent other requests from completing.
```rust,ignore
{{#include ../../examples/09_04_concurrent_tcp_server/src/main.rs:main_func}}
```
# Serving Requests in Parallel
Our example so far has largely presented concurrency (using async code)
as an alternative to parallelism (using threads).
However, async code and threads are not mutually exclusive.
In our example, `for_each_concurrent` processes each connection concurrently, but on the same thread.
The `async-std` crate allows us to spawn tasks onto separate threads as well.
Because `handle_connection` is both `Send` and non-blocking, it's safe to use with `async_std::task::spawn`.
Here's what that would look like:
```rust
{{#include ../../examples/09_05_final_tcp_server/src/main.rs:main_func}}
```
Now we are using both concurrency and parallelism to handle multiple requests at the same time!
See the [section on multithreaded executors](../08_ecosystem/00_chapter.md#single-threading-vs-multithreading)
for more information.
