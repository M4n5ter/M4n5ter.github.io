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

订阅之后，我们在返回的 subscriber 上调用 [`into_stream()`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Subscriber.html#method.into_stream) 。这个方法会消费掉这个 `Subscriber` ，返回一个能在消息到达的时候生成消息的 stream 。在我们开始迭代消息之前，注意这个 stream 通过[`tokio::pin!`](https://docs.rs/tokio/1/tokio/macro.pin.html) 被  [pin](https://doc.rust-lang.org/std/pin/index.html) 到了栈上。 
