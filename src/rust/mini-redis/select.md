## Select

目前为止，我们想要为系统增加并发的话，我们需要生成一个新的任务。我们现在将介绍一些其它的方式来使用 Tokio 并发的执行异步代码。

## `tokio::select!`

这个 `tokio::select!` 宏允许在多个异步计算上等待并且在**单个**计算完成时返回。

举个例子：

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        let _ = tx1.send("one");
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

使用了两个 oneshot channel，它俩中的任何一个都可以第一个完成。`select!` 语句会在这两个 channel 上 await ，并且绑定 `val` 到任务的返回值上。当 `tx1` 或 `tx2` 完成，相关联的 block 会被执行。

**没有**完成的分支会被直接 drop 掉。在这个例子中，计算会 await 在每个 channel 的 `oneshot::Receiver` 上。没有完成的 `oneshot::Receiver` 会被丢弃。

### Cancellation

在异步 Rust 中，取消表现为 drop 一个 future。回想一下 "[Async in depth](https://m4n5ter.github.io/rust/mini-redis/async_in_depth.html)"，异步 Rust 操作通过 future 实现，并且 future 是惰性的。只有 future 被 poll 了，才会有新的进展。如果 future 被 drop 了，那么相关联的状态也会被 drop ，也就是说不会再有新的进展了。

也就是说，有时异步操作会产生后台任务或启动在后台运行的其他操作。举个例子，再上面的示例中，一个任务被创建用来在背后发送消息，一般来说，任务将会执行一些计算来生成值。

Futures 或者其它类型可以实现 `Drop` 来清理背后的资源。Tokio 的 `oneshot::Receiver` 通过往 `Sender` 发送一个关闭信号来实现 `Drop` 。这个收到关闭信号的 sender 会通过 drop 来中断正在执行的操作。

```rust
use tokio::sync::oneshot;

async fn some_operation() -> String {
    // Compute value here
}

#[tokio::main]
async fn main() {
    let (mut tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        // Select on the operation and the oneshot's
        // `closed()` notification.
        tokio::select! {
            val = some_operation() => {
                let _ = tx1.send(val);
            }
            _ = tx1.closed() => {
                // `some_operation()` is canceled, the
                // task completes and `tx1` is dropped.
            }
        }
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

### The `Future` implementation

为了更好的理解 `select!` 的工作方式，让我们看一下假设的 `Future` 实现会是什么样子。这是一个简化版本，在实践中，`select!` 包含了其它的功能，例如随机选择第一个被 poll 的分支。

```rust
use tokio::sync::oneshot;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct MySelect {
    rx1: oneshot::Receiver<&'static str>,
    rx2: oneshot::Receiver<&'static str>,
}

impl Future for MySelect {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if let Poll::Ready(val) = Pin::new(&mut self.rx1).poll(cx) {
            println!("rx1 completed first with {:?}", val);
            return Poll::Ready(());
        }

        if let Poll::Ready(val) = Pin::new(&mut self.rx2).poll(cx) {
            println!("rx2 completed first with {:?}", val);
            return Poll::Ready(());
        }

        Poll::Pending
    }
}

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    // use tx1 and tx2

    MySelect {
        rx1,
        rx2,
    }.await;
}
```

这个 `MySelect` future 包含了每个分支的 future。当 `MySelect` 被 poll，第一个分支会被 pol，如果它就绪了，它返回出来的 val 就会被用掉，并且 `MySelect` 会立即结束。在 `.await` 从 future 收到输出后，future 会被 drop 掉，这导致 future 内的两个分支也会被 drop 。因为有一个分支没有完成，这个分支的操作实际上被取消了。

记住上一节的内容：

> 当一个 future 返回 `Poll::Pending` ，它**必须**确保 waker 是在某一点被注册了。忘记这么做会导致任务被无限期地挂起

在这个 `MySelect` 实现中，没有显式的使用 `Context` 参数。相反，这个 waker 要求通过在内部传递 `cx` 给内部的 future 满足了。因为内部的 future 也必须满足 waker 要求，通过仅在从内部 future 接收到 `Poll::Pending` 时返回 `Poll::Pending` 来满足，所以 `MySelect` 也满足了 waker 要求。（用我的理解就是 `MySelect` 靠内部的分支返回 `Poll::Ready` 时它也返回 `Poll::Ready` ，内部分支返回 `Poll::Pending` 时它也返回 `Poll::Pending` 来隐式的满足了上面引用中的要求 ）

## Syntax（语法）

这个 `select!` 宏可以处理多于两个分支的情况，目前的限制是 64 个分支（可以通过在宏里继续多处理一些分支 ，但是因为 64 个分支已经够多了，一直再宏里增加分支上限也不优雅）。每个分支像这样构成：

```rust
<pattern> = <async expression> => <handler>,
```

当 `select` 宏被执行，所有的 `<async expression>` 会被聚合起来，然后并发执行。当有一个表达式率先完成，表达式的结果会被匹配到 `<pattern>` 。如果结果匹配了模式，那么所有剩余的 `<async expression>` 会被 drop 掉，并且完成了的那个表达式的 `<handler>` 被执行。`<handler>` 表达式可以访问 `<pattern>` 建立的任何绑定。

`<pattern>` 最基本的情况就是一个变量名，`<async expression>` 的结果会被绑定到这个变量名，并且 `<handler>` 能访问这个变量。这就是为什么最开始的例子里在 `<pattern>` 和 `<handler>` 被使用的 `val` 是访问的 `<async expression>` 的 `val` 。

如果 `<pattern>` **没有**成功匹配异步计算的结果，那么剩下的异步表达式继续并发执行，直到出现下一个先执行完的 `<async expression>` 。然后相同的逻辑会继续应用到结果上，以此类推。

因为 `select!` 可以携带任何异步表达式，所以在 select 上定义更加复杂的计算变得有可能了。

这里，我们 select 一个 `oneshot` 输出个一个 TCP connection。

```rust
use tokio::net::TcpStream;
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    // Spawn a task that sends a message over the oneshot
    tokio::spawn(async move {
        tx.send("done").unwrap();
    });

    tokio::select! {
        socket = TcpStream::connect("localhost:3465") => {
            println!("Socket connected {:?}", socket);
        }
        msg = rx => {
            println!("received message first {:?}", msg);
        }
    }
}
```

这里，我们 select 一个 oneshot 和从 `TcpListener` 接收 sockets 。

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        tx.send(()).unwrap();
    });

    let mut listener = TcpListener::bind("localhost:3465").await?;

    tokio::select! {
        _ = async {
            loop {
                let (socket, _) = listener.accept().await?;
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {}
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
```

这个 accept loop 会一直运行直到遇到 error 或者 `rx` 收到一个值。`_` 模式表示我们对异步计算返回的值并不感兴趣。



## Return value

`tokio::select!` 宏会返回 `<handler>` 表达式计算出的结果。

```rust
async fn computation1() -> String {
    // .. computation
}

async fn computation2() -> String {
    // .. computation
}

#[tokio::main]
async fn main() {
    let out = tokio::select! {
        res1 = computation1() => res1,
        res2 = computation2() => res2,
    };

    println!("Got = {}", out);
}
```

因为这个，它要求**每个**分支的 `<handler>` 表达式计算出同样的类型。如果 `select!` 的输出不被需要，一个不错的实践是让表达式返回 `()`



## Errors

使用 `?` 操作符从表达式传播错误。它如何工作取决于 `?` 是从异步表达式还是从 handler 使用。在异步表达式中使用 `?` 把错误从异步表达式中传播出去，这会使这个异步表达式的输出变成 `Result` 。在 handler 中使用 `?` 会立即将错误传播到 `select!` 表达式外部。让我们再来看看这个 accpet loop ：

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    // [setup `rx` oneshot channel]

    let listener = TcpListener::bind("localhost:3465").await?;

    tokio::select! {
        res = async {
            loop {
                let (socket, _) = listener.accept().await?;
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {
            res?;
        }
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
```

请关注 `listener.accept().await?` 。这个 `?` 操作符把错误传播出了 `<async expression>` 并且绑定到了 `res` 。发生错误时 `res` 会被设置成 `Err(_)` ，然后在 handler 中，`?` 操作符再次被使用，`res?` 语句会把错误传播出 `main` 函数。



## Pattern matching

回顾一下 `select!` 宏的分支语法定义：

```rust
<pattern> = <async expression> => <handler>,
```

目前为止，我们仅仅在 `<pattern>` 上使用了变量绑定。然而，任何 Rust 模式都可以被使用，举个例子，如果说我们从多个 MPSC channels 接收，我们可能会这么做：

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (mut tx1, mut rx1) = mpsc::channel(128);
    let (mut tx2, mut rx2) = mpsc::channel(128);

    tokio::spawn(async move {
        // Do something w/ `tx1` and `tx2`
    });

    tokio::select! {
        Some(v) = rx1.recv() => {
            println!("Got {:?} from rx1", v);
        }
        Some(v) = rx2.recv() => {
            println!("Got {:?} from rx2", v);
        }
        else => {
            println!("Both channels closed");
        }
    }
}
```

在这个例子中， `select!` 表达式等待从 `rx1` 和 `rx2` 接收一个 value 。如果一个 channel 关闭了，`recv()` 会返回 `None` ，这将**无法**匹配例子中的模式，并且当前分支会被禁用。这个 `select!` 表达式将会继续在剩余的分支上 wait 。

请注意例子中的 `select!` 表达式包含一个 `else` 分支。这个 `select!` 表达式必须计算出一个 value，当使用模式匹配，可能**没有**一条分支能成功匹配它们所关联的模式，如果这种情况发生了， `else` 分支就会被计算。



## Borrowing

当生成任务时，生成的异步表达式必须拥有它里面的数据的所有权。但是 `select!` 宏没有这个限制，每条分支的异步表达式可能是**借用**的数据并且进行并发操作。遵循 Rust  的借用规则，多个异步表达式可以一起**不可变借用**单个数据或者单个异步表达式可以**可变借用**单个数据。

让我们看一下几个例子。这里，我们同时发送相同的数据到两个不同的 TCP 目标。

```rust
use tokio::io::AsyncWriteExt;
use tokio::net::TcpStream;
use std::io;
use std::net::SocketAddr;

async fn race(
    data: &[u8],
    addr1: SocketAddr,
    addr2: SocketAddr
) -> io::Result<()> {
    tokio::select! {
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr1).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr2).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        else => {}
    };

    Ok(())
}
```

这里的 `data` 变量是被下面的两个异步表达式**不可变借用**的。当其中一个操作成功完成，另一个将会被 drop。因为我们使用 `Ok(_)` 来模式匹配，如果一个表达式失败了，另一个会继续执行。

当来到每条分支的 `<handler>` ，`select!` 保证只有单个 `<handler>` 会运行。正因如此，每个 `<handler>` 可以**不可变借用**相同的数据。

下面这个例子在两个 handler 中都对 `out` 进行了修改：

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    let mut out = String::new();

    tokio::spawn(async move {
        // Send values on `tx1` and `tx2`.
    });

    tokio::select! {
        _ = rx1 => {
            out.push_str("rx1 completed");
        }
        _ = rx2 => {
            out.push_str("rx2 completed");
        }
    }

    println!("{}", out);
}
```



## Loops

`select!` 宏经常被用在循环中。这一部分将会通过几个示例来展示在循环中使用 `select!` 宏的常见方式。我们通过 select 多个 channels 开始：

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx1, mut rx1) = mpsc::channel(128);
    let (tx2, mut rx2) = mpsc::channel(128);
    let (tx3, mut rx3) = mpsc::channel(128);

    loop {
        let msg = tokio::select! {
            Some(msg) = rx1.recv() => msg,
            Some(msg) = rx2.recv() => msg,
            Some(msg) = rx3.recv() => msg,
            else => { break }
        };

        println!("Got {:?}", msg);
    }

    println!("All channels have been closed.");
}
```

这个例子在三个 channel 的 rx 上select。当从任意一个 channel 中接收到一条消息，会将其打印到 STDOUT 。当有一条 channel close 了， `recv()` 会返回 `None` ，通过使用模式匹配，`select!` 宏会继续在剩余的 channels 上等待。当所有的 channel 都 close 了，这里的 `else` 分支会被计算，并且循环会终止。

`select!` 宏随机选中分支来先检查一下是否就绪。当多个 channel 挂起了它们的 value，一个随机的 channel 会被选中，并从它上面接收值。这是为了处理 receive loop 处理消息的速度比消息被推进 channel 的速度慢的情况，意味着 channel 开始被填满了。如果 `select!` **没有**随机选中一个分支来第一个检查，那么每次循环在迭代的时候， `rx1` 都会被第一个检查，如果 `rx1` 总是持有新消息，那么剩余的 channel 永远都不会被检查。

> 如果当 `select!` 被计算时，多个 channel 都有挂起的值，只有一个 channel 能有值被 pop 出去。所有其它的 channel 保持未被接触（没轮到它们），并且它们的消息会留在 channel 中直到下一次循环迭代。不会有消息丢失。

### Resuming an async operation（恢复异步操作）

现在我们将会展示如何跨多个 `select!` 调用运行异步操作。在这个例子中，我们有一个 消息类型是 `i32` 的 MPSC channel ，还有一个异步函数。我们希望运行这个异步函数直到它完成或者一个偶数从 channel 中被接收。

```rust
async fn action() {
    // Some asynchronous logic
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);    
    
    let operation = action();
    tokio::pin!(operation);
    
    loop {
        tokio::select! {
            _ = &mut operation => break,
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    break;
                }
            }
        }
    }
}
```

请注意，与在 `select!` 宏内调用 `action()` 不同，它在循环**外部**被调用。`action()` 的返回值被分配到了 `operation` 且**没有**调用 `.await` 。然后我们对 `operation` 调用了 `tokio::pin!` 。

在 `select!` 循环内，与传入 `operation` 不同，我们传入了 `&mut operation` 。这个 `operation` 变量正在跟踪执行中的异步操作。每次循环迭代使用这个相同的 operation 而不是来一次新的 `action()` 调用。

`select!` 的另一个分支从 channel 接收消息，如果消息是一个偶数，我们就结束循环。否则，再次开始 `select!` 

这是我们第一次使用 `tokio::pin!` ，我们还没打算深挖这它的细节。只需要注意，对一个引用进行 `.await` 调用，这个被引用的值必须被 pin 或者实现了 `Unpin` 。

如果我们移除 `tokio::pin!` 这一行，并且尝试编译，我们会得到以下错误：

```rust
error[E0599]: no method named `poll` found for struct
     `std::pin::Pin<&mut &mut impl std::future::Future>`
     in the current scope
  --> src/main.rs:16:9
   |
16 | /         tokio::select! {
17 | |             _ = &mut operation => break,
18 | |             Some(v) = rx.recv() => {
19 | |                 if v % 2 == 0 {
...  |
22 | |             }
23 | |         }
   | |_________^ method not found in
   |             `std::pin::Pin<&mut &mut impl std::future::Future>`
   |
   = note: the method `poll` exists but the following trait bounds
            were not satisfied:
           `impl std::future::Future: std::marker::Unpin`
           which is required by
           `&mut impl std::future::Future: std::future::Future`
```

尽管我们已经在 [Async in depth](https://m4n5ter.github.io/rust/mini-redis/async_in_depth.html) 了解了 `Future` ，但是这个错误对我们来说仍然不是很清晰。如果你在尝试对一个**reference** 调用 `.await` 时碰到了这样的一个关于 " `Future` 没有实现... " 的错误，那么这个 future 可能需要被 pin 。

从 [standard library](https://doc.rust-lang.org/std/pin/index.html) 阅读更多关于 [`Pin`](https://doc.rust-lang.org/std/pin/index.html) 的细节。

### Modifying a branch

让我们看一个稍微复杂一些的 loop。我们有：

1. 一个内容是 `i32` 的 channel。

2. 一个对 `i32` 执行的异步操作。

我们想要实现的逻辑是：

1. 在 channel 上等待一个**偶数**

2. 使用这个偶数作为输入来开始一个异步操作。

3. 等待这个异步操作，但是同时要从 channel 监听更多的偶数。

4. 如果一个新的偶数在已经存在的异步操作完成之前被接收到了，退出存在的异步操作并且用新的偶数再跑一个。

```rust
async fn action(input: Option<i32>) -> Option<String> {
    // If the input is `None`, return `None`.
    // This could also be written as `let i = input?;`
    let i = match input {
        Some(input) => input,
        None => return None,
    };
    // async logic here
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);
    
    let mut done = false;
    let operation = action(None);
    tokio::pin!(operation);
    
    tokio::spawn(async move {
        let _ = tx.send(1).await;
        let _ = tx.send(3).await;
        let _ = tx.send(2).await;
    });
    
    loop {
        tokio::select! {
            res = &mut operation, if !done => {
                done = true;

                if let Some(v) = res {
                    println!("GOT = {}", v);
                    return;
                }
            }
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    // `.set` is a method on `Pin`.
                    operation.set(action(Some(v)));
                    done = false;
                }
            }
        }
    }
}
```

我们使用了和之前那个例子相似的方法。这个异步函数在循环外被调用，并且分配到 `operation` 。这个 `operation` 变量被 pin 了。这个循环会在 `operation` 和 channel receiver 上 select 。

请注意 `action` 是如何携带 `Option<i32>` 作为一个参数的。在我们接收到第一个偶数之前，我们需要实例化一个 `operation` 。我们使 `action` 携带 `Option` 并且返回 `Option` 。如果 `None` 被传递进去了，会返回  `None` 。第一次循环迭代， `operation` 会立即完成并返回 `None` （因为我们实例化它的时候穿的是 `None`）。

这个例子使用了一些新语法。这第一个分支包括 `,if !done` ，这是一个分支先决条件。在解释它是如何工作的之前，让我们看下如果省略这个先决条件会发生什么。移除 `,if !done` 并且运行例子会导致以下输出：

```rust
thread 'main' panicked at '`async fn` resumed after completion', src/main.rs:1:55
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

当尝试在 `operation` 已经完成**之后**使用它时，这个错误发生了。一般来说，当使用 `.await` ，这个被 await 的值就被消费掉了。在这个例子中，我们 await 了一个引用，这意味着 `operation` 在它完成后仍然存在。

为了避免这个 panic，如果 `operation` 已经完成，我们必须小心地禁用第一条分支。这里的 `done` 变量被用来跟踪 `operation` 是否完成。一个 `select!` 分支可能会包含一个 **precondition** （先决条件），这个先决条件会在 `select!` await 当前分支**之前**被检查（虽然顺序上它被写在后面，但是不影响它是一个 "先决条件"）。如果先决条件计算出了 `false` 那么该分支会被禁用。`done` 变量被初始化为 `false` 。当 `operation` 完成，`done` 会被设置为 `true` ，这样在下次循环迭代的时候将会禁用 `operation` 分支。当一个偶数消息从 channel 被接收，`operation` 会被重置，并且 `done` 会被设置为 `false` 。



## Per-task concurrency

`tokio::spawn` 和 `select!` 都能够运行并发的异步操作。然而，运行并发操作的策略有所不同。`tokio::spawn` 函数携带一个异步操作，并且生成一个新的任务去运行它。一个任务是 Tokio runtime 调度的对象。两个不同的任务会被 Tokio 独立调度，它们可能会同时运行在不同的操作系统线程上。正因如此，一个被生成的任务和被生成的线程具有相同的限制：不能有借用！

`select!` 宏会在**同一个任务**中并发运行所有的分支。因为所有的 `select!` 分支都在同一个任务中被执行，它们永远不可能**同时**被运行。`select!` 宏会在单个任务上多路复用异步操作。
