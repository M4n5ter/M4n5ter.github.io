## Async in depth （深入异步）

至此，我们已经完成了一个相当全面的异步 Rust 和 Tokio 之旅。现在我们将会深挖 Rust 的异步运行时模型。在本教程的开始，我们就提到了 异步 Rust 用了一种独一无二的方法。现在我们来解释一下是啥意思。



## Futures

作为快速回顾，我们来举一个非常基本的异步函数。与教程到目前为止所涵盖的内容相比，这并不是什么新鲜事。

```rust
use tokio::net::TcpStream;

async fn my_async_fn() {
    println!("hello from async");
    let _socket = TcpStream::connect("127.0.0.1:3000").await.unwrap();
    println!("async TCP operation complete");
}
```

我们调用了这个函数，并且返回了某个值，对这个值调用 `.await`。

```rust
#[tokio::main]
async fn main() {
    let what_is_this = my_async_fn();
    // Nothing has been printed yet.
    what_is_this.await;
    // Text has been printed and socket has been
    // established and closed.
}
```

`my_async_fn()` 返回的值是一个 future ，future 是一个实现了标准库提供的  [`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 的值。它们是包含正在进行的异步计算的值。

 [`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 的定义如下：

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Self::Output>;
}
```

关联类型( [associated type](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types) ) `Output` 是 future 一旦完成后会产生的类型。可以通过看标准库文档（[standard library](https://doc.rust-lang.org/std/pin/index.html)）得到更多细节。

不像其它语言实现的 future ，一个 Rust 的 future 不是代表一个正在后台发生的计算，而是 Rust future 就是计算本身。future 的所有者负责通过 poll the future 来推动计算，这就是 `Future::poll` 所做的事。

### Implementing `Future` （实现 `Future`）

让我们实现一个简单的 future。这个 future 将会：

1. 一直 wait 到特定时刻。

2. 输出一些文本到 STDOUT 。

3. 产生一个字符串。

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
}
```

### Async fn as a Future （异步函数作为 future）

在 main 函数中，我们实例化一个 future 并对它调用 `.await` 。在异步函数中，我们可以对任何实现了 `Future` 的值调用 `.await` 。相反，调用一个 `async` function 返回一个实现了 `Future` 的匿名类型。`async fn main()` 所生成的 future 类似于：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
    // Initialized, never polled
    State0,
    // Waiting on `Delay`, i.e. the `future.await` line.
    State1(Delay),
    // The future has completed.
    Terminated,
}

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use MainFuture::*;

        loop {
            match *self {
                State0 => {
                    let when = Instant::now() +
                        Duration::from_millis(10);
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => {
                    match Pin::new(my_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            *self = Terminated;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```
