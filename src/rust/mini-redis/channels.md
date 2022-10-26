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

如果你需要一个多生产者多消费者的 channel，其中每条消息只能由所有现有消费者中的一个接收，那么你可以使用  [`async-channel`](https://docs.rs/async-channel/) crate。异步 Rust 之外还有同步的 channel，比如 [`std::sync::mpsc`](https://doc.rust-lang.org/stable/std/sync/mpsc/index.html) 和 [`crossbeam::channel`](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html)。这些 channel 都会在等待消息的时候阻塞线程，这意味着它们不适合用在异步代码中。

在这块内容里，我们会使用 [mpsc](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html) 和 [oneshot](https://docs.rs/tokio/1/tokio/sync/oneshot/index.html) 。其他类型的 channel 会在之后的内容中探索。本节内容的完整代码在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs) 。



## Define the message type （定义消息类型）

在许多使用消息传递的场景下，接收消息的任务会响应多条命令。在我们的场景下，任务将会响应 `GET` 和 `SET` 命令。为了模拟这个，我们先定义一个 `Command` enum 。

```rust
use bytes::Bytes;

#[derive(Debug)]
enum Command {
    Get {
        key: String,
    },
    Set {
        key: String,
        val: Bytes,
    }
}
```



## Create the channel （创建通道）

在 `main` 函数中，我们创建一个 `mpsc` channel。

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // 创建一个新的 mpsc ，并给它的最大容量设置为 32。
    let (tx, mut rx) = mpsc::channel(32);

    // ... Rest comes here
}
```

`mpsc` 用来**发送**命令给管理 redis connection 的任务。多生产者的容量允许消息可以从多个任务中发送。创建 channel 会返回两个值，一个 sender（习惯上命名为 `tx`） 和一个 receiver （习惯上命名为 `rx`）。这俩句柄是分开使用的，它们可能会被移动到不同的任务中去。

这里的 channel 创建时指定了 32 个容量。如果消息发的比收的快，那么 channel 会把没来得及被接收的消息存起来。一旦 channel 中的 32 个位置都被消息填满了，这时候再调用 `send(...).await` 将会 sleep 直到有 1 个消息被 receiver 拿走去消费。

从多个任务发送消息是通过 clone `Sender` 做到的。例如：

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send("sending from first handle").await;
    });

    tokio::spawn(async move {
        tx2.send("sending from second handle").await;
    });

    while let Some(message) = rx.recv().await {
        println!("GOT = {}", message);
    }
}
```

两条消息都被发送到了单个 `Receiver` 句柄。在 `mpsc` channel 中克隆 receiver 是不被允许的。

当每个 `Sender` 超出作用域或者因为其他原因被 drop 了，就不再能往这个 channel 发送更多消息了。此时，在 `Receiver` 上调用 `recv` 将会返回 `None`，这意味着所有的 sender 都不在了，channel 被关闭了。 

在我们的场景下，管理 redis connection 的任务知道一旦 channel 被关闭，就得关闭 redis connection，因为 connection 不会再被使用了。



## Spawn manager task （生成管理者任务）

接下来，生成一个任务来处理来自 channel 的消息。首先，一个对 redis 的客户端连接会被建立。然后，受到的命令会通过 redis connection 被发送。

```rust
use mini_redis::client;
// The `move` keyword is used to **move** ownership of `rx` into the task.
let manager = tokio::spawn(async move {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Start receiving messages
    while let Some(cmd) = rx.recv().await {
        use Command::*;

        match cmd {
            Get { key } => {
                client.get(&key).await;
            }
            Set { key, val } => {
                client.set(&key, val).await;
            }
        }
    }
});
```

现在，更新这两个任务以通过通道发送命令，而不是直接在Redis连接上发出它们。

```rust
// The `Sender` handles are moved into the tasks. As there are two
// tasks, we need a second `Sender`.
let tx2 = tx.clone();

// Spawn two tasks, one gets a key, the other sets a key
let t1 = tokio::spawn(async move {
    let cmd = Command::Get {
        key: "hello".to_string(),
    };

    tx.send(cmd).await.unwrap();
});

let t2 = tokio::spawn(async move {
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
    };

    tx2.send(cmd).await.unwrap();
});
```

在 `main` 函数的底部，我们 `.await` 这些 [`JoinHandle`](https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html) 来确保commands 能够在进程退出前完全完成。

```rust
t1.await.unwrap();
t2.await.unwrap();
manager.await.unwrap();
```

## Receive responses （接收响应）

最后一步是从管理器任务接收响应(response)。`GET` command 需要获取 value 并且 `SET` command 需要知道它的操作是否成功完成。

为了传递响应，我们使用一个 `oneshot` channel。`oneshot` channel 是一个单生产者，单消费者的 channel，针对发送单一值进行了优化。在我们的场景下，响应就是单一值。

与 `mpsc` 类似，`oneshot::channel()` 返回一个 sender 和一个 receiver 句柄。

```rust
use tokio::sync::oneshot;

let (tx, rx) = oneshot::channel();
```

不像 `mpsc` ，`oneshot` 不需要指定容量，因为它的容量始终是 1。另外，`oneshot` 的两个句柄都不能被 clone。

为了从管理器任务接收响应，在发送一个 command 之前，要先创建一个 `oneshot` channel。`oneshot` 的 `Sender` 会被包含在发给管理器任务中的 command 中。而 `Receiver` 用来接收管理器任务用 `oneshot` 的 `Sender` 发送的消息。

首先，改变 `Command` 来包含 `Sender` 。方便起见，用了一个类型别名来使用 `Sender`。

```rust
use tokio::sync::oneshot;
use bytes::Bytes;

/// Multiple different commands are multiplexed over a single channel.
#[derive(Debug)]
enum Command {
    Get {
        key: String,
        resp: Responder<Option<Bytes>>,
    },
    Set {
        key: String,
        val: Bytes,
        resp: Responder<()>,
    },
}

/// Provided by the requester and used by the manager task to send
/// the command response back to the requester.
type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
```

现在，改变发送 command 的任务，让它包含一个 `oneshot::Sender`。

```rust
let t1 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Get {
        key: "hello".to_string(),
        resp: resp_tx,
    };

    // Send the GET request
    tx.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});

let t2 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
        resp: resp_tx,
    };

    // Send the SET request
    tx2.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});
```

在 `oneshot::Sender` 上的 `send` 调用是立即完成的，**不需要**一个 `.await` 。这是因为 `oneshot` channel 上的 `send` 总是立即返回 succeed 或者 fail ，而不需要任何形式的等待。

当接收端被 drop 时，往一个 oneshot channel 发送一个值会返回 `Err` 。这表示接收端不再对结果感兴趣了。在我们的假设中，接收端(想发命令的任务)不再对 response(管理器任务返回的结果) 感兴趣的情况是可接受的。所以通过 `resp.send(...)`  返回的 `Err`  就没必要处理了。

可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs)看到完整代码。



## Backpressure and bounded channels (背压和有界的通道)

这里的小标题我不会翻译 :(

每当引入并发(cibcurrency)和队列(queuing)的时候，确保队列有界且系统能优雅的处理负载是非常重要的。无界的队列将会导致可用内存耗尽，并且还会导致系统陷入无法预测的失败中。

Tokio 会注意避免隐式队列。事实上很大一部分是因为异步操作是惰性的（这在前面提到过，这也是 rust 与其它实现 `async/await` 的语言的不同之处）。思考下下面的情况：

```rust
loop {
    async_op();
}
```

如果异步操作迫切的希望被运行，loop 循环在没有确保先前的操作完成的情况下，反复将新的 `async_op` 排进一个队列来运行，这会导致隐式的无界队列。基于回调（callback）和基于勤奋 future（rust 是惰性 future）的系统会特别容易受到这种影响。

然而~，使用 Tokio 和异步 Rust ，上述片段根本就不会被运行。这是因为 `.await` 从未被调用。如果上述片段改成使用 `.await` ，那么这个循环就会在重新开始之前等待操作执行完毕。

```rust
loop {
    // 在 `async_op` 完成之前是不会重新开始循环的
    async_op().await;
}
```

并发和队列必须被显式地引入。这么做的方法包括：

* `tokio::spawn`

* `select!`

* `join!`

* `mpsc::channel`

当需要这么做的时候，请确保并发的总量是有界的（不要无限制的创建 task）。举个例子，当写一个 TCP accept loop 的时候，确保打开的 socket 总数是有界的。当使用 `mpsc::channel`时，选择一个能够被管理的容量限度（容量不要超出实际承受能力）。指定有界值是特定于应用的。

小心和选择好的界限是编写可靠的Tokio应用程序的重要组成部分。
