## Spawning

我们接下来准备开始完成我们的 Redis server！

首先，把上一部分的客户端的 `SET / GET` 代码移动到一个示例文件中去，这样我们可以在 server 上去运行它。

```zsh
mkdir -p examples
mv src/main.rs examples/hello-redis.rs
```

创建一个新的空的 `src/main.rs` 后再继续。



## Accepting sockets（从 sockets 接收）

英语水平有限，这小标题只能翻译成这样了 :(

首先我们的 Redis server 第一件需要做的事情就是**接受入站的 TCP sockets**。用 **`tokio::net::TcpListener`** 来完成。

> Tokio 的许多类型用了与 Rust 标准库中的等价的同步类型一样的名字。并且 Tokio 使用 `async fn` 暴露了与 **`std`** 相同的 `APIs` 

一个 `TcpListener` 绑定在 **6379** 端口，接着 socket 们会在一个 loop 中被接受。每个 socket 都会被处理然后关闭。现在，为门将要读取命令，然后打印它到标准输出，并且回复一个 error。

```rust
use mini_redis::{Connection, Frame};
use tokio::net::{TcpListener, TcpStream};

#[tokio::main]
async fn main() {
    // 绑定 listener 到一个地址
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // 解构出来的第二个 item 包含一个新 connection 的一对 IP 和 port，这里将其忽略了
        let (socket, _) = listener.accept().await.unwrap();
        process(socket).await;
    }
}

async fn process(socket: TcpStream) {
    // `Connection` 让我们能够读写 redis **frames**(抽象的帧) 而不是
    // byte streams(字节流). `Connection` 类型由 mini-redis 定义。
    let mut connection = Connection::new(socket);

    if let Some(frame) = connection.read_frame().await.unwrap() {
        println!("GOT: {:?}", frame);

        // Respond with an error
        let response = Frame::Error("unimplemented".to_string());
        connection.write_frame(&response).await.unwrap();
    }
}

```

现在把它跑起来：

```zsh
cargo run
```

在另一个终端窗口，运行 `hello-redis` example（上一节我们写的那个 `SET / GET`）

```zsh
cargo run --example hello-redis
```

输出应该得是像下面这样：

```zsh
Error: "unimplemented"
```

在跑服务端的那个终端，输出应该是下面这样：

```zsh
GOT: Array([Bulk(b"set"), Bulk(b"hello"), Bulk(b"world")])
```



## 并发

我们的 server 有一个问题（除去只回复了错误）。它一次只会处理一个入站请求：当一个连接被接受，我们的 server 停留在 accept loop 里面，直到 `response` 被完全写入 socket。

我们肯定是希望我们的 Redis server 能够处理并发的请求，为了达到这个目的，我们需要加并发。

> 并发(concurrency)和并行(parallelism)不是一回事。如果一个线程交替执行两个任务，那么就是同时(CPU 有能力让你感觉到是“同时”，尽管同一时间点一个线程只可能在处理一个任务)处理这两个任务(这是并发)，但不是并行处理。要让这变成并行，那么至少需要 2 个线程，每个线程都执行一个任务。
> 
> 使用 `Tokio` 的优点之一是异步代码允许您同时处理许多任务，而不必使用普通线程并行处理它们。事实上，Tokio可以在单个线程上同时运行许多任务！



为了并发处理这些连接，对每个入站连接都得生成一个新任务，连接会在这个新任务中被处理。

accept loop 会变成这样：

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    // 绑定 listener 到一个地址
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // 解构出来的第二个 item 包含一个新 connection 的一对 IP 和 port，这里将其忽略了
        let (socket, _) = listener.accept().await.unwrap();
        // 生成一个新任务，socket 的所有权被移动到了这个新任务里面，并在那里被处理。
        tokio::spawn(async move {
            process(socket).await;
        });
    }
}
```

### Tasks

一个 Tokio 任务是一个异步的 green thread。他们是通过 `async` 块传递给 `tokio::spawn`  来创建的。`tokio::spawn` 函数返回一个 `JoinHandle`，使得 `JoinHandle`  的调用者可以与生成的任务进行交互。`async` 块可以拥有返回值，调用者通过在 `JoinHandle` 上使用 `.await` 来获取返回值。

举个栗子：

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // 在这里做了一些异步的事情
        "return value"
    });

    // 又做了一些别的事情

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```

`.await` 会让出当前线程的控制权，并等待 `JoinHandle` 返回一个 `Result`。当一个任务在执行期间遇到了一个错误，`JoinHandle` 将会返回一个 `Err` ，当任务 panic 又或者是因为 `runtime` 关闭而被强制取消也会发生前面那个事件。

Task 是由 scheduler 管理的执行单位。Spawn (生产) 一个任务会把任务提交给 Tokio scheduler来确保任务在有工作要做时执行。生产出来的任务可能会在它们被生产的线程上执行，也有可能会在不一样的 `runtime thread` 上被执行。任务被生产后也能够在不同线程间移动。

任务在 Tokio 中是非常非常轻量的。在底层，它们只需要一次分配和64字节的内存。应用程序应该可以随意生成数千甚至数百万个任务。



## `'static` bound（静态生命周期绑定）

当我们在 Tokio runtime 上生成了一个任务，其类型的生命周期必须是 `'static`。这意味着生成的任务不得包含对任务外部拥有的数据的任何引用。

> 一个常见的错觉是：`'static` 总是意味着 "永远存活"，但事实并非如此。仅仅因为一个值是静态的并不意味着你有内存泄漏。想知道更多可以看这里 [Common Rust Lifetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program) 。

举个不能被编译通过的例子:D

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

尝试编译它会有如下报错：

```rust
error[E0373]: async block may outlive the current function, but
              it borrows `v`, which is owned by the current function
 --> src/main.rs:7:23
  |
7 |       task::spawn(async {
  |  _______________________^
8 | |         println!("Here's a vec: {:?}", v);
  | |                                        - `v` is borrowed here
9 | |     });
  | |_____^ may outlive borrowed value `v`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:7:17
  |
7 |       task::spawn(async {
  |  _________________^
8 | |         println!("Here's a vector: {:?}", v);
9 | |     });
  | |_____^
help: to force the async block to take ownership of `v` (and any other
      referenced variables), use the `move` keyword
  |
7 |     task::spawn(async move {
8 |         println!("Here's a vec: {:?}", v);
9 |     });
  |
```

这种情况会发生是因为默认情况下，变量不会被 **move** 到 async block。这个 `v`  Vector 被 `main` 函数保留了所有权。`println!` 只是借用了 `v`。rust 编译器向我们解释了这一点，甚至提出了修复建议！（rust 编译器还是一如既往的牛逼！尽管它的严格经常会让我很挫败:( ）

按 rust 编译器说的来，在第 7 行处为 async block 加上 `move` ，现在这个 task 就拥有了 v 的所有权而不是借用，并且让它变成了 `'static`。
