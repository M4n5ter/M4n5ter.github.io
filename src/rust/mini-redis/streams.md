## Streams

stream 是一种异步的一系列的值。它的异步等效与 Rust 的 [`std::iter::Iterator`](https://doc.rust-lang.org/book/ch13-02-iterators.html) 并且通过 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) trait 来表示。Streams 能够在 `async` functions 中被迭代。它们也可以通过适配器被转换。Tokio 在 [`StreamExt`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html) trait 上提供了一些场景的适配器。

Tokio 在一个单独的 crate 提供了 stream 支持：`tokio-stream` 。

```toml
tokio-stream = "0.1"
```

> 目前，Tokio 的 stream 实用工具存在与 `tokio-strean` crate 中。一旦 `Stream` trait 在 Rust 标准库中稳定了，Tokio 的 stream 将会被移动到 `tokio` crate 。

## Iteration

目前为止，Rust 编程语言没有支持异步的 `for` 循环。作为替代，迭代 stream 通过使用  [`StreamExt::next()`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.next) 并配对一个 `while let` 循环来完成。

```rust
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let mut stream = tokio_stream::iter(&[1, 2, 3]);

    while let Some(v) = stream.next().await {
        println!("GOT = {:?}", v);
    }
}
```

像迭代器一样，`next()` 方法返回 `Option<T>` ，T 是 stream 里的值的类型。接收到 `None` 表示这个 stream iteration 终止了。

### Mini-Redis broadcast （Mini-Redis 广播）

让我们使用 Mini-Redis client 来回顾一个稍微更复杂的例子。

完整的代码可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/streams/src/main.rs)找到。

```rust
use tokio_stream::StreamExt;
use mini_redis::client;

async fn publish() -> mini_redis::Result<()> {
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Publish some data
    client.publish("numbers", "1".into()).await?;
    client.publish("numbers", "two".into()).await?;
    client.publish("numbers", "3".into()).await?;
    client.publish("numbers", "four".into()).await?;
    client.publish("numbers", "five".into()).await?;
    client.publish("numbers", "6".into()).await?;
    Ok(())
}

async fn subscribe() -> mini_redis::Result<()> {
    let client = client::connect("127.0.0.1:6379").await?;
    let subscriber = client.subscribe(vec!["numbers".to_string()]).await?;
    let messages = subscriber.into_stream();

    tokio::pin!(messages);

    while let Some(msg) = messages.next().await {
        println!("got = {:?}", msg);
    }

    Ok(())
}

#[tokio::main]
async fn main() -> mini_redis::Result<()> {
    tokio::spawn(async {
        publish().await
    });

    subscribe().await?;

    println!("DONE");

    Ok(())
}
```

一个任务被生成来发布消息到 Mini-Redis server 上的 "numbers" channel 。然后，在主要的任务中，我们订阅 "numbers" channel 并且打印接收到的消息。

订阅之后，我们在返回的 subscriber 上调用 [`into_stream()`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Subscriber.html#method.into_stream) 。这个方法会消费掉这个 `Subscriber` ，返回一个能在消息到达的时候生成消息的 stream 。在我们开始迭代消息之前，注意这个 stream 通过[`tokio::pin!`](https://docs.rs/tokio/1/tokio/macro.pin.html) 被  [pin](https://doc.rust-lang.org/std/pin/index.html) 到了栈上。 在一个 stream 上调用 `next()` 要求这个 stream 是 [pinned](https://doc.rust-lang.org/std/pin/index.html) （这也是上面用 `tokio::pin!` 的原因）。`into_stream()` 函数返回一个没有被 pin 的 stream，为了迭代这个 stream ，我们必须显式地 pin 它。

> 当一个 Rust 的值不再能够在内存中被移动时，这个值就是 "pinned" 。a pinned value 的关键属性是指针可以取到 pinned data 并且调用者可以确信指针是有效的。这个特性被 `async/await` 用来支持跨 `.await` 点的借用数据。

如果我们忘记 pin the stream，我们会得到一个像这样的错误：

```rust
error[E0277]: `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>` cannot be unpinned
  --> streams/src/main.rs:29:36
   |
29 |     while let Some(msg) = messages.next().await {
   |                                    ^^^^ within `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`, the trait `Unpin` is not implemented for `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>`
   |
   = note: required because it appears within the type `impl Future`
   = note: required because it appears within the type `async_stream::async_stream::AsyncStream<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 'static)>>, impl Future>`
   = note: required because it appears within the type `impl Stream`
   = note: required because it appears within the type `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because it appears within the type `tokio_stream::map::_::__Origin<'_, tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because it appears within the type `tokio_stream::take::_::__Origin<'_, tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::take::Take<tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
```

如果你遇到了类似这样的错误信息，请尝试 pin 这个值！！！

在尝试运行这个之前，先把 Mini-Redis server 跑起来：

```zsh
mini-redis-server
```

然后尝试运行上面的代码。我们将会看到消息被输出到了 STDOUT。

```textile
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"four" })
got = Ok(Message { channel: "numbers", content: b"five" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

由于订阅和发布之间存在竞争，一些早期消息可能会被删除。该程序永远不会退出。只要服务器处于活动状态，对 Mini-Redis 频道的订阅就会保持活动状态。

让我们看看可以怎么来用 stream 拓展这个程序。

## Adapters （适配器）

接收一个 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) 并且返回另一个 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) 的函数通常被称为 'stream adapters' ，因为它们是 'adapter pattern' 的一种形式。常见的 stream adapters 包括 [`map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.map)、 [`take`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.take) 和 [`filter`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter) 。

让我们更新一下 Mini-Redis 来让它能够退出。在接收到 3 个消息后，停止迭代消息，使用 [`take`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.take) 来完成这个目的。这个 adapter 限制 stream 生产**至多** `n` 条消息（n 条消息后 `while let` 就拿不到 `Some` 了，程序就能退出了）。

```rust
let messages = subscriber
    .into_stream()
    .take(3);
```

再次运行程序，我们得到了：

```textile
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
```

这次程序结束了。

现在，让我们把 stream 限制为个位数，我们将会通过检查消息的长度来确保此事。我们使用 [`filter`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter) adapter 来 drop 任何不匹配先决条件的消息。

```rust
let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .take(3);
```

再次运行程序，我们得到：

```textile
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

请注意，adapter 的应用顺序很重要。先调用 `filter` 然后 `take` 是跟先 `take` 然后 `filter` 不一样的（这很好理解，先 `take` 的话，就会在前三个里找内容是个位数的消息）。

最后，我们将通过剥离 `Ok(Message{...})` 部分来整理输出，这通过 [`map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.map) 来完成。因为这是在 `filter` 之后被应用的，我们能知道消息是 `Ok`，所以我们可以使用 `unwrap()` 。

```rust
let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .map(|msg| msg.unwrap().content)
    .take(3);
```

现在，输出是：

```textile
got = b"1"
got = b"3"
got = b"6"
```

另一种选择是使用 [`filter_map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter_map) 将 [`filter`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter) 和 [`map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.map) 两个步骤组合起来作为一个单次调用。

在[这里](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html)可以找到更多可用的 adapter。

## Implementing `Stream`

 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) trait 和 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 非常类似。

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}
```

`Stream::poll_next()` 函数非常像 `Future::poll` ，除了它可以被反复调用来从 stream 接收许多值。正如我们在[Async in depth](https://m4n5ter.github.io/rust/mini-redis/async_in_depth.html)了解到的一样，当一个 stream **没有**准备好返回一个值的时候，`Poll::Pending` 会被返回。任务的 waker 会被注册，一旦 stream 应该被再次 poll 的时候，waker 会被通知。

这里的 `size_hint()` 方法的使用方式跟 [iterators](https://doc.rust-lang.org/book/ch13-02-iterators.html) 里的一样，它会返回 stream 剩余长度是上下界，`(0,` [`None`](https://doc.rust-lang.org/nightly/core/option/enum.Option.html#variant.None "None")`)` 是它的默认实现，这对任何 stream 来说都是正确的。

通常来说，当手动实现一个 `Stream` 的时候，它是通过组合 future 和其它 stream 来完成的。作为一个示例，让我们重建在[Async in depth](https://m4n5ter.github.io/rust/mini-redis/async_in_depth.html)实现的 `Delay` future，我们将会把它转换成一个以 10 ms 为间隔，生成 3 次 `()` 的 stream 。

```rust
use tokio_stream::Stream;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

struct Interval {
    rem: usize,
    delay: Delay,
}

impl Stream for Interval {
    type Item = ();

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<()>>
    {
        if self.rem == 0 {
            // No more delays
            return Poll::Ready(None);
        }

        match Pin::new(&mut self.delay).poll(cx) {
            Poll::Ready(_) => {
                let when = self.delay.when + Duration::from_millis(10);
                self.delay = Delay { when };
                self.rem -= 1;
                Poll::Ready(Some(()))
            }
            Poll::Pending => Poll::Pending,
        }
    }
}
```

### `async-stream`

手动用 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) trait 来实现 stream 是非常冗长乏味的。不幸的是，Rust 编程语言还不支持 `async/await` 来定义 stream 。这项工作正在做，但是还没就绪。

[`async-stream`](https://docs.rs/async-stream) crate 可以作为一个临时解决方案使用，这个 crate 提供了一个 `stream!` 宏，它能将输入转化成一个 stream。通过使用这个 crate，上面的 interval 可以像这样被实现：

```rust
use async_stream::stream;
use std::time::{Duration, Instant};

stream! {
    let mut when = Instant::now();
    for _ in 0..3 {
        let delay = Delay { when };
        delay.await;
        yield ();
        when += Duration::from_millis(10);
    }
}
```
