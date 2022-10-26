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
