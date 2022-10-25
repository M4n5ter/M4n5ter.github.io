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

## Concurrency （并发）

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

如果必须同时从多个任务访问单个数据，那么就必须使用 `Arc` 等同步原语共享它。

下面引用的内容我觉得比较难理解：

> Note that the error message talks about the argument type *outliving* the `'static` lifetime. This terminology can be rather confusing because the `'static` lifetime lasts until the end of the program, so if it outlives it, don't you have a memory leak? The explanation is that it is the *type*, not the *value* that must outlive the `'static` lifetime, and the value may be destroyed before its type is no longer valid.
> 
> When we say that a value is `'static`, all that means is that it would not be incorrect to keep that value around forever. This is important because the compiler is unable to reason about how long a newly spawned task stays around. We have to make sure that the task is allowed to live forever, so that Tokio can make the task run as long as it needs to.
> 
> The article that the info-box earlier links to uses the terminology "bounded by `'static`" rather than "its type outlives `'static`" or "the value is `'static`" to refer to `T: 'static`. These all mean the same thing, but are different from "annotated with `'static`" as in `&'static T`.

留意关于**参数类型**的寿命超过了 `’static` 生命周期的错误信息。这个术语可能会让人很困惑，因为 `'static` 生命周期将会一直存在直到程序结束，所以如果比它寿命还长，确定没有内存泄漏吗？ 关于这个的解释是：它是一个类型，而不是一个必须寿命长过 `'static`' 的值，并且它的值可能会在它的类型失效之前被销毁。

当我们说一个值是 `'static` 的时候，这意味着永远留着它常常是正确的。这非常重要，因为编译器无法推断新生成的任务会保留多长时间。我们不得不确保任务被允许一直存活（仅仅是允许，但不是必须），这样 Tokio 就可以让任务运行它实际需要的时间。

前面的信息框链接到的文章使用术语 **“以 `'static` 为界”** 而不是 **“其类型的寿命超过 `'static` ”** 或 **“其值是 `'static`"** 来指代 `T：'static`。这些都意味着同一件事，但不同于 `&‘static T` 中的 **“用 `'static` 注释”**

插一句嘴：上面这块儿我是琢磨了很久，但是还有一些内容没完全明白，看来还是有待提升呐～

## `Send` bound

从 `tokio::spawn` 生成的任务**必须**实现 `Send` trait 。这样才能当任务被 `.await` 后允许 Tokio runtime 在线程之间移动他们。

因为水平有限，可能有误，所以附上原文后再给出我的理解：

> Tasks are `Send` when **all** data that is held **across** `.await` calls is `Send`. This is a bit subtle. When `.await` is called, the task yields back to the scheduler. The next time the task is executed, it resumes from the point it last yielded. To make this work, all state that is used **after** `.await` must be saved by the task. If this state is `Send`, i.e. can be moved across threads, then the task itself can be moved across threads. Conversely, if the state is not `Send`, then neither is the task.

当一个任务内所有跨过 `.await` 调用的数据都实现了 `Send` 时，这个任务才是实现了 `Send` 的。如下例子就会因为 `a` 没有实现 `Send` 且跨过了 `.await` 调用而导致编译失败：

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    // 绑定 listener 到一个地址
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // 解构出来的第二个 item 包含一个新 connection 的一对 IP 和 port，这里将其忽略了
        let (socket, _) = listener.accept().await.unwrap();
        let a = Rc::new("Rc does not impl Send");
        // 生成一个新任务，socket 的所有权被移动到了这个新任务里面，并在那里被处理。
        tokio::spawn(async move {
            process(socket).await;
            println!("{:?}", a);
        });
    }
}
```

```rust
error: future cannot be sent between threads safely
   --> src/main.rs:16:9
    |
16  |         tokio::spawn(async move {
    |         ^^^^^^^^^^^^ future created by async block is not `Send`
    |
    = help: within `impl std::future::Future<Output = ()>`, the trait `std::marker::Send` is not implemented for `std::rc::Rc<&str>`
note: captured value is not `Send`
   --> src/main.rs:18:30
    |
18  |             println!("{:?}", a);
    |                              ^ has type `std::rc::Rc<&str>` which is not `Send`
note: required by a bound in `tokio::spawn`
   --> /home/m4n5ter/.cargo/registry/src/rsproxy.cn-8f6827c7555bfaf8/tokio-1.21.2/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ^^^^ required by this bound in `tokio::spawn`

error: could not compile `my-redis` due to previous error
```

因为用了 `.await` 后，当前任务会让出线程控制权，任务的当前状态会被整个打包，并且可能会在多个线程间传递这个任务，存在任务会在不同的线程被执行的可能，而数据在线程间传递要求实现 `Send` trait 。

下面是官方给出的两个例子：

成功：

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // The scope forces `rc` to drop before `.await`.
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` is no longer used. It is **not** persisted when
        // the task yields to the scheduler
        yield_now().await;
    });
}
```

失败：

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` is used after `.await`. It must be persisted to
        // the task's state.
        yield_now().await;

        println!("{}", rc);
    });
}
```

错误报告：

```rust
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
6   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    | 
   ::: [..]spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```

我们会在下一节 Shared state 来更深入的探讨这个错误的特殊情况。

## Store values（存储值）

我们现在将要实现 `process` 函数来处理发送过来的命令。我们使用 `HashMap` 来存储值。`SET` 命令将会插入数据到 `HashMap` ，`GET` 值会加载数据。另外，我们将会使用一个 loop 来接受每个连接的多个命令。

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

        // 生成一个新任务，socket 的所有权被移动到了这个新任务里面，并在那里被处理。
        tokio::spawn(async move { process(socket).await });
    }
}

async fn process(socket: TcpStream) {
    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // 一个 `HashMap` 用来存储数据
    let mut db = HashMap::new();

    // `Connection` 让我们能够读写 redis **frames**(抽象的帧) 而不是
    // byte streams(字节流). `Connection` 类型由 mini-redis 定义。
    let mut connection = Connection::new(socket);

    // 使用 `read_frame` 来从`connection`接收一个`Command`。
    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                // 值被存储为 `Vec<u8>`
                db.insert(cmd.key().to_string(), cmd.value().to_vec());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                if let Some(value) = db.get(cmd.key()) {
                    // `Frame::Bulk` 期望数据是`Bytes` 类型的。
                    // 这个类型将会在教程的后面部分讨论。
                    // 现在`&Vec<u8>` 通过 `into()` 被转换成了 `Bytes` 。
                    Frame::Bulk(value.clone().into())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        // Write the response to the client
        connection.write_frame(&response).await.unwrap();
    }
}
```

让我们来试一试：

```zsh
cargo run
```

另一个终端窗口执行：

```zsh
cargo run --example hello-redis
```

出现了如下输出：

```zsh
从服务端得到了值; result=Some(b"world")
```

我们现在可以获取和设置值，但是有一个问题：这些值在连接之间不共享。如果另一个套接字连接并尝试获取hello键，它将找不到任何东西。

在下一节中，我们将为所有套接字实现持久化数据。
