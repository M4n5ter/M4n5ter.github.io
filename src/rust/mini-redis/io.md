## I/O

Tokio 的 I/O 操作大致与 `std` 中的相同，但是是异步的。这有一个为读取而生的 trait [`AsyncRead`](https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html) 和一个为写入而生的 trait [`AsyncWrite`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html) 。一些特定的类型恰当的实现了这些 trait（[`TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html), [`File`](https://docs.rs/tokio/1/tokio/fs/struct.File.html), [`Stdout`](https://docs.rs/tokio/1/tokio/io/struct.Stdout.html)）。[`AsyncRead`](https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html) 和 [`AsyncWrite`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html) 也被一些像 `Vec<u8>` 和 `&[u8]` 这样的数据结构实现了。这允许在需要 reader 或 writer 的地方使用字节数组。

本章将会覆盖基础的 Tokio I/O 读写并且通过几个例子来说明。下一章将会给出一个更加高级的 I/O 示例。

## `AsyncRead` and `AsyncWrite`

这两个 trait 提供了异步读写字节流的工具。在这些 trait 上的方法通常不会直接调用，就好像你不会手动从 `Future` 调用 `poll` 方法。相反，我们都是通过 [`AsyncReadExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html) and [`AsyncWriteExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html) 提供的实用方法来使用它们。

让我们简略的看一下它俩的几个方法。这些方法都是 `async` ，所以都必须用 `.await` 来使用。

### `async fn read()`

[`AsyncReadExt::read`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read) 提供了一个异步方法来读取数据到一个 buffer，返回读取的字节数。

**Note：** 当 `read()` 返回了 `Ok(0)` ，这标志着 stream 关闭了。任何对 `read()` 的进一步调用都会立即返回 `Ok(0)` 。对 [`TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html) 实例来说，这标志着 socket 的 the read half 关闭了。

```rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer[..]).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

### `async fn read_to_end()`

[`AsyncReadExt::read_to_end`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read_to_end) 会从 stream 读取所有的字节直到 EOF。

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = Vec::new();

    // read the whole file
    f.read_to_end(&mut buffer).await?;
    Ok(())
}
```

### `async fn write()`

[`AsyncWriteExt::write`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write) 把一个 buffer 写入到 writer，返回写入的字节数。

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    // Writes some prefix of the byte string, but not necessarily all of it.
    let n = file.write(b"some bytes").await?;

    println!("Wrote the first {} bytes of 'some bytes'.", n);
    Ok(())
}
```

### `async fn write_all()`

[`AsyncWriteExt::write_all`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all) 把整个 buffer 写入 writer，与上面那个不一样，这哥们就不返回写入的字节数了。

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    file.write_all(b"some bytes").await?;
    Ok(())
}
```

这两个特征都包括许多其他有用的方法。有关完整的方法列表，请参阅API文档。

## Helper functions （辅助函数）

此外，就像 `std`， [`tokio::io`](https://docs.rs/tokio/1/tokio/io/index.html) 模块包含了一些有用的工具函数以及用于处理 [standard input](https://docs.rs/tokio/1/tokio/io/fn.stdin.html)、 [standard output](https://docs.rs/tokio/1/tokio/io/fn.stdout.html) 和 [standard error](https://docs.rs/tokio/1/tokio/io/fn.stderr.html) 的API。例如，[`tokio::io::copy`](https://docs.rs/tokio/1/tokio/io/fn.copy.html) 异步的将 reader 的全部内容 copy 到一个 writer 。

```rust
use tokio::fs::File;
use tokio::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut reader: &[u8] = b"hello";
    let mut file = File::create("foo.txt").await?;

    io::copy(&mut reader, &mut file).await?;
    Ok(())
}
```

请注意，这种用法体现了 `&[u8]` 也实现 `AsyncRead` 的事实。

## Echo server （回声服务）

让我们做些玩意儿来练习下异步I/O。我们将要写一个回声服务。

这个回声服务要绑定在一个 `TcpListener` 并且在一个 loop 中接收入站连接。对每个入站连接来说，数据从 socket 中读取并立即写回 socket。客户端发送数据到服务端，并接收回相同的数据。

我们将会用两种不同的方案来实现两次回声服务。

### Using `io::copy()`

开始，我们将用 [`io::copy`](https://docs.rs/tokio/1/tokio/io/fn.copy.html) 实用工具来实现 echo 逻辑。

你可以写在一个新的 binary 文件中：

```zsh
touch src/bin/echo-server-copy.rs
```

可以通过以下方式启动（或只是检查编译）：

```zsh
cargo run --bin echo-server-copy
```

我们能够使用一个标准的命令行工具，比如 `telnet` 来测试我们的回声服务，或者通过写一个简单的客户端，就像在 [`tokio::net::TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#examples) 文档中找到的那个一样。

这是一个 TCP server 并且需要一个 accept loop。一个新的任务被生成来处理每个接收到的 socket 。

```rust
use tokio::io;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            // Copy data here
        });
    }
}
```

就像前面说的，这个工具函数接收一个 reader 参数和一个 writer 参数，并且将数据从一个 copy 到另一个中。然而啊，我们只有一个 `TcpStream` ，这单个值同时实现了 `AsyncRead` 和 `AsyncWrite` 。可是由于 `io::copy` 对 reader 和 writer 都要求 `&mut` ，这 socket 不能同时作为放到这两个参数上。

```rust
// 这是无法编译的
io::copy(&mut socket, &mut socket).await
```

### Splitting a reader + writer

为了解决这个难题，我们必须把 socket 分离成一个 reader 句柄和一个 writer 句柄。拆分 reader/writer 组合的最佳方法是使用 [`io::split`](https://docs.rs/tokio/1/tokio/io/fn.split.html)。

任何同时实现了 reader + writer 的类型都能够使用 [`io::split`](https://docs.rs/tokio/1/tokio/io/fn.split.html) 实用工具来拆分。这个函数接收单个的值并返回分离的 reader 和 writer 句柄。这两个句柄可以被独立使用，包括分别在两个单独的任务中使用。

举个例子，echo 客户端可以像这样并发处理读写：

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> io::Result<()> {
    let socket = TcpStream::connect("127.0.0.1:6142").await?;
    let (mut rd, mut wr) = io::split(socket);

    // Write data in the background
    tokio::spawn(async move {
        wr.write_all(b"hello\r\n").await?;
        wr.write_all(b"world\r\n").await?;

        // 有时候，rust 的类型推断器需要一点点的帮助
        Ok::<_, io::Error>(())
    });

    let mut buf = vec![0; 128];

    loop {
        let n = rd.read(&mut buf).await?;

        if n == 0 {
            break;
        }

        println!("GOT {:?}", &buf[..n]);
    }

    Ok(())
}
```

因为 `io::split` 支持**任何**实现了 `AsyncRead + AsyncWrite` 的值，并返回独立的句柄，`io::split` 在内部使用了一个 `Arc` 和 一个 `Mutex` （这意味着会有蛮大的开销）。如果 socket 是 `TcpStream` 的情况就能避免这种开销。`TcpStream` 提供了两个专门的函数（[`TcpStream::split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.split) 和 [`into_split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.into_split)）。

[`TcpStream::split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.split) 接收一个 `&mut TcpStream` 并返回一个 reader 和 一个 writer 句柄。正因为使用的是引用，所以这两个句柄必须跟 `split()` 调用待在**同一**任务中。虽然有前面这个限制，但是它的这种专门实现是**零开销**的，没有 `Arc` 也没有 `Mutex` 。`TcpStream` 也提供了 [`into_split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.into_split) 来支持处理可跨任务使用的场景，开销缩减到了只有一个 `Arc`。

因为 `io::copy()` 调用是跟持有 `TcpStream` 的任务是同一个任务（跟上面那段代码中的情况不同，上面的代码的 rd 跟 wr 在不同的任务中），这就意味着我们完全可以使用 [`TcpStream::split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.split) 。在 server 处理 echo 逻辑的任务变成了下面这样：

```rust
tokio::spawn(async move {
    let (mut rd, mut wr) = socket.split();

    if io::copy(&mut rd, &mut wr).await.is_err() {
        eprintln!("failed to copy");
    }
});
```

可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server-copy.rs)找到完整代码。

### Manual copying （手动 copy）

现在，来看一下我们要如何通过手动 copy data 来写 echo server。为了做到这点，我们使用 [`AsyncReadExt::read`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read) 和 [`AsyncWriteExt::write_all`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all) 。

完整的 server 代码是这样：

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = vec![0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    // Return value of `Ok(0)` signifies that the remote has
                    // closed
                    Ok(0) => return,
                    Ok(n) => {
                        // Copy the data back to socket
                        if socket.write_all(&buf[..n]).await.is_err() {
                            // Unexpected socket error. There isn't much we can
                            // do here so just stop processing.
                            return;
                        }
                    }
                    Err(_) => {
                        // Unexpected socket error. There isn't much we can do
                        // here so just stop processing.
                        return;
                    }
                }
            }
        });
    }
}
```

（你可以把这段代码放到 `src/bin/echo-server.rs`  并用 `cargo run --bin echo-server` 启动它）

我是 arch linux ：

```zsh
yay -S netcat
echo 你好 | nc 127.0.0.1 6142
```

让我们分析一下：首先，因为使用了  `AsyncRead` 和 `AsyncWrite` ，所以 extension traits （`AsyncReadExt` 和`AsyncWriteExt`）必须被引入。 

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
```

### Allocating a buffer （申请缓冲区）

这种策略是为了从 socket 读取一些数据到缓冲区，然后再把缓冲区的内容写回 socket。

```rust
let mut buf = vec![0;1024];
```

显式地避免了栈上缓冲区。回顾一下[之前](https://m4n5ter.github.io/rust/mini-redis/spawning.html#send-bound) ，我们注意到所有的跨 `.await` 调用的数据都得由任务本身存储。而在这个场景， `buf` 被用来跨 `.await` 。所有的任务数据被存储在同一个内存块。你可以把它想象成一个 `enum` ，`enum` 内的变量都是需要为一个特定的 `.await` 存储的数据。

如果这个 `buf` 是一个栈数组，每个被生成的用来接受 socket 的任务的内部结构可能看起来会像这样：

```rust
struct Task {
    // internal task fields here
    task: enum {
        AwaitingRead {
            socket: TcpStream,
            buf: [BufferType],
        },
        AwaitingWriteAll {
            socket: TcpStream,
            buf: [BufferType],
        }

    }
}
```

如果一个栈数组被用来当做 buffer type，它将会被内联在任务结构体中。这会导致任务结构体非常庞大。另外，缓冲区大小通常是 page size (*Modern hardware and software tend to load data into RAM (and transfer data from RAM to disk) in discrete chunk called pages*)。这反过来又会使任务的大小变得尴尬：`\$page-size + 几个字节`。

Linus 有一篇吐槽贴说:

> Just do the math. I've done it. 4kB is good. 8kB is borderline ok. 16kB or more is simply not acceptable.
> 
> [Real World Technologies - Forums - Thread: Cache pipeline](https://www.realworldtech.com/forum/?threadid=144991&curpostid=145006)

所以 linux 的 page size 应该会控制在 16kB 以内。

编译器优化 async blocks 的布局比优化一个 basic `enum` 要多很多。实际上，变量不会像 `enum` 所要求的那样在枚举变体之间移动。但是，任务结构体的大小至少与最大变量一样大。

正因如此，为 buffer 使用一个专门的内存分配通常是更有效的（这里是 `Vector`）。

### Handling EOF （处理 EOF）

当 TCP stream 读的那一半句柄关闭了，再去调用 `read()` 会返回 `Ok(0)` 。在这种时候退出 read loop 是很重要的。忘记在 EOF 的时候退出 read loop 是一个常见的 bug 来源。

```rust
loop {
    match socket.read(&mut buf).await {
        // Return value of `Ok(0)` signifies that the remote has
        // closed
        Ok(0) => return,
        // ... other cases handled here
    }
}
```

忘记退出 read loop 通常会导致 100% CPU占用的无限循环。这是因为 socket 关闭后，`socket.read()` 会立即返回，循环就会永远的重复下去。

完整代码看[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server.rs)
