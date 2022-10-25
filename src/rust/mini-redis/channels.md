## Channels TODO

现在我们已经了解了一些关于 Tokio 的并发，让我们把它们应用到客户端侧吧。把我们先前写的服务端的代码移动到一个显式的二进制文件里去：

```zsh
mkdir src/bin
mv src/main.rs src/bin/server.rs
```

然后创建一个新的 binary 来放我们的客户端代码：

```rust
touch src/bin/client.rs
```

在这个文件中，我们将会写关于本节的代码。无论何时你想运行它，请先启动 server 端：

```zsh
cargo run --bin server
```

然后在另一个终端窗口：

```zsh
cargo run --bin client
```

话都说到这个份上了，来让我们开始 code 吧！

比如说我门想要运行两个并发的 Redis commands。我们可以为每个 command 生成一个任务。然后两个命令就能并发啦～

一开始啊，我们可能会想到下面这种方式：

```rust
use mini_redis::client;

#[tokio::main]
async fn main() {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Spawn two tasks, one gets a key, the other sets a key
    let t1 = tokio::spawn(async {
        let res = client.get("hello").await;
    });

    let t2 = tokio::spawn(async {
        client.set("foo", "bar".into()).await;
    });

    t1.await.unwrap();
    t2.await.unwrap();
}
```

不幸的是呢，编译器阻止了我们继续，因为两个任务都需要用某种方式访问 `client` 。由于

`Client` 并没有实现 `Copy` trait ，所以如果没有一些代码来促成 `client` 的共享是不能被编译通过的。再说，`Client::set` 需要 `&mut self` ，这意味着调用它的时候需要独占 `Client` 的访问。我们可以为每个连接打开一个任务，但是这并不理想。因为 `.await` 需要带着锁被调用，所以我们不能使用 `std::sync::Mutex` 。我们可以使用 `tokio::sync::Mutex` ，但是这会导致同一时间只能有一个请求（即 singleflight 单飞）。如果客户端实现了 [pipelining](https://redis.io/topics/pipelining) ，一个异步锁会导致连接的低利用率。



## Message passing （消息传递）

实践答案是使用消息传递！这种模式包含生成一个专门的任务来管理 `client` 资源。任何想要发起请求的任务都要发送消息给这个 `client` 任务。`client` 任务的角色相当于代理人，它会代表发送者(sender)来发送请求(request)，并把响应(response)发回给发送者(sender)。

采用这种策略，需要创建一个单独的连接。管理 `client` 的任务能够独占访问权限以便调用 `set` 和 `get` 。此外， channel 以缓冲区的方式工作。当 `client` 任务正忙的时候，任务可能会被发送到 `client` 。一旦 `client` 空闲了，可以处理新请求了，它会从 channel 拉去下一个请求。这种方式可以有更好的吞吐量，并且能够被拓展，支持连接池。



## Tokio's channel primitives （Tokio 的通道原语）

Tokio 提供了 [一些 channel](https://docs.rs/tokio/1/tokio/sync/index.html) ，每个都有不一样的目的。

* [mpsc](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html)：多生产者，单消费者的 channel。可以发送许多值。

* [oneshot](https://docs.rs/tokio/1/tokio/sync/oneshot/index.html)：单生产者，单消费者的 channel。可以发送单个值。

* [broadcast](https://docs.rs/tokio/1/tokio/sync/broadcast/index.html)：多生产者，多消费者。可以发送许多值，每个接收者都能看到每个值。

* [watch](https://docs.rs/tokio/1/tokio/sync/watch/index.html)：单生产者，多消费者。可以发送许多值，但是不会保留历史值。接收者只能看到最新的值。
