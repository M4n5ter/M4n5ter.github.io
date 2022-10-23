## Hello Tokio

我们将从编写一个非常基本的 `Tokio` 应用程序开始。它将连接到 `Mini-Redis` 服务器，设置 `key` `hello` 的值为 `world` 。然后它将读回 `key`。这将使用 `Mini-Redis` 的客户端库完成。

## 代码

---

### 生成一个新的 `crate`

```zsh
cargo new my-redis
cd my-redis
```

### 添加依赖

接下来打开 `Cargo.toml` 并把下面的内容添加到 `[dependencies]` 下：

```toml
tokio = { version = "1", features = ["full"] }
mini-redis = "0.4"
```

### 写代码

然后打开 `main.rs` 并将文件的内容替换成下面的：

```rust
use mini_redis::{client, Result};

#[tokio::main]
async fn main() -> Result<()> {
    // 向 mini-redis 的地址打开一个连接.
    let mut client = client::connect("127.0.0.1:6379").await?;

    // 设置一个叫 `hello` 的 key，它的内容是 `world`
    client.set("hello", "world".into()).await?;

    // 去 get 这个 `hello`
    let result = client.get("hello").await?;

    println!("从服务端得到了值; result={:?}", result);

    Ok(())
}
```

确保 `Mini-Redis server` 正在运行，找个单独的终端窗口执行:

```zsh
mini-redis-server
```

现在，让我们运行我门的 `my-redis` 应用程序。

```zsh
❯ cargo run         
    Finished dev [unoptimized + debuginfo] target(s) in 0.15s
     Running `target/debug/my-redis`
从服务端得到了值; result=Some(b"world")
```

这样便是成功了，也算是即将要开始 coding 了！

## 看看具体发生什么

让我们回顾一下刚刚做的事情，代码量不多，但是其实发生了很多事情。

```rust
let mut client = client::connect("127.0.0.1:6379").await?;
```

`client::connect` 函数是由 `mini_redis` 这个 crate 提供的。它通过**异步**的方式向指定的地址建立一个 TCP 连接。一旦连接成功建立了，将会返回一个 `Client` handle(中文叫句柄)（这里给它起了个名 "client"）。

即使这个操作是**异步**执行的，但是我们写的这个代码看起来像是**同步**的。通过 `.await` 操作符来表明这是一个异步操作。

### 何为异步编程？

相信看过 `The book` 或者 `Rust course` 的大伙都知道，下面就贴原文啦～

> Most computer programs are executed in the same order in which they are written. The first line executes, then the next, and so on. With synchronous programming, when a program encounters an operation that cannot be completed immediately, it will block until the operation completes. For example, establishing a TCP connection requires an exchange with a peer over the network, which can take a sizeable amount of time. During this time, the thread is blocked.
> 
> With asynchronous programming, operations that cannot complete immediately are suspended to the background. The thread is not blocked, and can continue running other things. Once the operation completes, the task is unsuspended and continues processing from where it left off. Our example from before only has one task, so nothing happens while it is suspended, but asynchronous programs typically have many such tasks.
> 
> Although asynchronous programming can result in faster applications, it often results in much more complicated programs. The programmer is required to track all the state necessary to resume work once the asynchronous operation completes. Historically, this is a tedious and error-prone task.

当然还有机翻可供粗略观摩:

> 大多数计算机程序都是按照它们编写的顺序执行的。第一行执行，然后是下一行，依此类推。使用同步编程，当程序遇到不能立即完成的操作时，它会阻塞，直到操作完成。例如，建立传输控制协议需要通过网络与对等方进行交换，这可能需要相当长的时间。在此期间，线程被阻塞。
> 对于异步编程，不能立即完成的操作会被挂起到后台。线程不会被阻塞，并且可以继续运行其他事情。一旦操作完成，任务就会被取消挂起，并从它停止的地方继续处理。我们之前的示例只有一个任务，所以挂起时什么都不会发生，但是异步程序通常有许多这样的任务。
> 虽然异步编程可以带来更快的应用程序，但它通常会导致更复杂的程序。一旦异步操作完成，程序员需要跟踪恢复工作所需的所有状态。从历史上看，这是一项乏味且容易出错的任务。

### 编译期的绿色线程(Compile-time green-threading)

>  `green-threading` 我的理解是一种非常轻量的“线程”，比如协程(`coroutine`)，以及直接被融入 `Go runtime` 的 `goroutine`（类似 coroutine，但又不同） 。

Rust 通过叫作 **`async/await`** 的特征来实现异步编程。执行异步操作的函数用 **`async`** 关键字来标记。在我们的示例中，connect函数是这样定义的：

```rust
use mini_redis::Result;
use mini_redis::client::Client;
use tokio::net::ToSocketAddrs;

pub async fn connect<T: ToSocketAddrs>(addr: T) -> Result<Client> {
    // ...
}
```

**`async fn`** 这样的定义方式看起来像是一个常规的同步函数，但是以异步的方式运行。

Rust 在**编译期将** **`async fn`** 转化为一个异步运行的 `routine` （不是 `coroutine`，不要理解错误）。

在 `async fn` 中对 `.await` 的任何调用都会将控制权返回给线程（即让出当前线程），此时这个操作会被放在后台，而线程可能会去做一些别的事情。

> 尽管也有其它语言实现了 `async/await` ，但 Rust 采用了一种独特的方法。
> 
> 大多情况下，Rust 的异步操作表现为 **`lazy`**，这导致了不同于其它语言的运行时语义。

如果还是不太明白，没有关系！我们将会在这整个教程中探索到更多关于 **`async/await`** 的知识。

### 使用 `async/await`

**异步函数**的调用与任何其他Rust函数一样。但是，调用这些函数不会导致函数体执行。换而言之，调用**异步函数**会返回一个代表这个操作的值（在概念上类似于一个没有参数的闭包）。

如果要真正地去执行这个操作，需要对这个返回值使用 `.await` 操作符。

就像下面这样：

```rust
async fn say_world() {
    println!("world");
}

#[tokio::main]
async fn main() {
    // 直接调用 `say_world()` 并不会执行它的函数体。
    let op = say_world();

    // 这个 println! 会先出现。
    println!("hello");

    // 对 `op` 调用 `.await`。
    op.await;
}
```

输出会是下面这样的：

```zsh
hello
world
```

`async fn` 的返回值是实现 `Future` trait的匿名类型。

`Future` 可以被看作是一个会在未来的某个时间点被执行的东西。

### 异步的 `main` 函数

`main` 函数与大多数的 Rust crate 不同，它被用来启动一个应用程序。

1. 它是一个 **`async fn`**

2. 它是用 `#[tokio::main]` 来注释的

当我们想进入一个异步的上下文，会使用 **`async fn`**。然而，异步函数必须被一个 **`runtime`** 所执行（tokio 就是 Rust 社区大名鼎鼎的异步运行时）。**`runtime`** 包括异步任务调度器、提供事件 I/O、计时器等。**`runtime`** 不会自动启动，所以 `main` 函数需要去启动它。

`#[tokio::main]` 是一个宏。它将 `async fn main()` 转化为一个同步`fn main()`，初始化了一个 `runtime` 实例并且执行了这个异步 main 函数。

例如以下内容：

```rust
#[tokio::main]
async fn main() {
    println!("hello");
}
```

被转化成：

```rust
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

`tokio runtime` 的细节将在后面介绍。

### Cargo features

在本教程引入 `tokio` 依赖时，`full` feature flag 被启用了。

```toml
tokio = { version = "1", features = ["full"] }
```

`Tokio` 有很多功能（`TCP`、`UDP`、`Unix sockets`、`timer`、`sync utilities`、`multiple scheduler types` 等）。并非所有应用程序都需要所有功能(`full`)。当尝试优化编译时间或最终应用程序占用空间时，应用程序可以决定只选择它用到的那些功能。

目前，我们在依赖 `tokio` 时使用 `full` feature，来方便 code。
