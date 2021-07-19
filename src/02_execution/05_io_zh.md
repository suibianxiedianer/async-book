# 执行器与系统 IO

在之前的 [`Future` 特征] 中，我们讨论了一个在套接字上进行异步读取的 future 示例：

```rust,ignore
{{#include ../../examples/02_02_future_trait/src/lib.rs:socket_read}}
```

这个 future 将从一个套接字中读取可用数据，当里面无数据时，
它就会将自身交还于执行器，以便其再次就绪、有数据可读时被唤醒。
但是，在这个例子中并不能清楚地了解到 `Socket` 类型是如何实现的，
尤其无法明确得知 `set_readable_callback` 函数是如何工作的。
一旦套接字就绪（可读），我们如何去安排调用 `wake()`？
to be called once the socket becomes readable? One option would be to have
a thread that continually checks whether `socket` is readable, calling
`wake()` when appropriate. However, this would be quite inefficient, requiring
a separate thread for each blocked IO future. This would greatly reduce the
efficiency of our async code.

In practice, this problem is solved through integration with an IO-aware
system blocking primitive, such as `epoll` on Linux, `kqueue` on FreeBSD and
Mac OS, IOCP on Windows, and `port`s on Fuchsia (all of which are exposed
through the cross-platform Rust crate [`mio`]). These primitives all allow
a thread to block on multiple asynchronous IO events, returning once one of
the events completes. In practice, these APIs usually look something like
this:

```rust,ignore
struct IoBlocker {
    /* ... */
}

struct Event {
    // An ID uniquely identifying the event that occurred and was listened for.
    id: usize,

    // A set of signals to wait for, or which occurred.
    signals: Signals,
}

impl IoBlocker {
    /// Create a new collection of asynchronous IO events to block on.
    fn new() -> Self { /* ... */ }

    /// Express an interest in a particular IO event.
    fn add_io_event_interest(
        &self,

        /// The object on which the event will occur
        io_object: &IoObject,

        /// A set of signals that may appear on the `io_object` for
        /// which an event should be triggered, paired with
        /// an ID to give to events that result from this interest.
        event: Event,
    ) { /* ... */ }

    /// Block until one of the events occurs.
    fn block(&self) -> Event { /* ... */ }
}

let mut io_blocker = IoBlocker::new();
io_blocker.add_io_event_interest(
    &socket_1,
    Event { id: 1, signals: READABLE },
);
io_blocker.add_io_event_interest(
    &socket_2,
    Event { id: 2, signals: READABLE | WRITABLE },
);
let event = io_blocker.block();

// prints e.g. "Socket 1 is now READABLE" if socket one became readable.
println!("Socket {:?} is now {:?}", event.id, event.signals);
```

Futures executors can use these primitives to provide asynchronous IO objects
such as sockets that can configure callbacks to be run when a particular IO
event occurs. In the case of our `SocketRead` example above, the
`Socket::set_readable_callback` function might look like the following pseudocode:

```rust,ignore
impl Socket {
    fn set_readable_callback(&self, waker: Waker) {
        // `local_executor` is a reference to the local executor.
        // this could be provided at creation of the socket, but in practice
        // many executor implementations pass it down through thread local
        // storage for convenience.
        let local_executor = self.local_executor;

        // Unique ID for this IO object.
        let id = self.id;

        // Store the local waker in the executor's map so that it can be called
        // once the IO event arrives.
        local_executor.event_map.insert(id, waker);
        local_executor.add_io_event_interest(
            &self.socket_file_descriptor,
            Event { id, signals: READABLE },
        );
    }
}
```

We can now have just one executor thread which can receive and dispatch any
IO event to the appropriate `Waker`, which will wake up the corresponding
task, allowing the executor to drive more tasks to completion before returning
to check for more IO events (and the cycle continues...).

[`Future` 特征]: ./02_future_zh.md
[`mio`]: https://github.com/tokio-rs/mio
