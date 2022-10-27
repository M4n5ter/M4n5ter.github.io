## Framing

我们接下来将会应用我们在 I/O 章节的所学，并实现 Mini-Redis 的框架层（framing layer，或许应该叫帧层） 。Framing 是获取 byte stream 并转化成 a stream of frames（帧） 的过程。一个 frame (帧) 是两个对等端（此处应该指代 client and server）之间传输数据的单位。Redis protocal frame 定义如下：

```rust
use bytes::Bytes;

enum Frame {
    Simple(String),
    Error(String),
    Integer(u64),
    Bulk(Bytes),
    Null,
    Array(Vec<Frame>),
}
```

注意 Frame 是如何包含没有任何语义的数据的， Command 解析和实现发生再更高级的层，而不在 Frame。

对于 HTTP 来说，一个 frame 可能看起来像这样：

```rust
enum HttpFrame {
    RequestHead {
        method: Method,
        uri: Uri,
        version: Version,
        headers: HeaderMap,
    },
    ResponseHead {
        status: StatusCode,
        version: Version,
        headers: HeaderMap,
    },
    BodyChunk {
        chunk: Bytes,
    },
}
```

为了实现 Mini-Redis 的 frame，我们将会实现一个 `Connecton` 结构来包装一个 `TcpStream` 和 reads/writes `mini_redis::Frame` values。

```rust
use tokio::net::TcpStream;
use mini_redis::{Frame, Result};

struct Connection {
    stream: TcpStream,
    // ... other fields here
}

impl Connection {
    /// Read a frame from the connection.
    /// 
    /// Returns `None` if EOF is reached
    pub async fn read_frame(&mut self)
        -> Result<Option<Frame>>
    {
        // implementation here
    }

    /// Write a frame to the connection.
    pub async fn write_frame(&mut self, frame: &Frame)
        -> Result<()>
    {
        // implementation here
    }
}
```

可以在[这里](https://redis.io/topics/protocol) 找到 Redis wire protocal 的细节。完整的 `Connection` 代码在[这里](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs) 。

## Buffered reads （带缓冲地读）

`read_frame` 方法在返回前会等待一个完整的 frame 被接收。单个 `TcpStream::read()` 调用可能会返回一个任意数量的数据。这个数据可能是一个完整的 frame、一个不完整 frame 或者多个 frame。如果接收到了一个不完整的 frame，数据会被放入 buffer 并且会继续从 socket 读更多数据。如果接收到了多个 frame，第一个帧会被返回，剩下的数据会被放入 buffer 直到下次 `read_frame` 调用。

为了实现这个， `Connection` 需要一个 read buffer 字段。数据从 socket 被读入这个 read buffer。当一个帧被解析，相对应的数据会从 buffer 中被移除。

我们将会用 `BytesMut` 作为 buffer type。它是一个可变版本的 `Bytes` 。

```rust
use bytes::BytesMut;
use tokio::net::TcpStream;

pub struct Connection {
    stream: TcpStream,
    buffer: BytesMut,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream,
            // Allocate the buffer with 4kb of capacity.
            buffer: BytesMut::with_capacity(4096),
        }
    }
}
```

下面，我们实现 `read_frame()` 方法。

```rust
use tokio::io::AsyncReadExt;
use bytes::Buf;
use mini_redis::Result;

pub async fn read_frame(&mut self)
    -> Result<Option<Frame>>
{
    loop {
        // Attempt to parse a frame from the buffered data. If
        // enough data has been buffered, the frame is
        // returned.
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // There is not enough buffered data to read a frame.
        // Attempt to read more data from the socket.
        //
        // On success, the number of bytes is returned. `0`
        // indicates "end of stream".
        if 0 == self.stream.read_buf(&mut self.buffer).await? {
            // The remote closed the connection. For this to be
            // a clean shutdown, there should be no data in the
            // read buffer. If there is, this means that the
            // peer closed the socket while sending a frame.
            if self.buffer.is_empty() {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        }
    }
}
```

让我们来分析一下它。 `read_frame` 方法操作了一个 loop。首先，`self.parse_frame()` 被调用。它会尝试从 `self.buffer` 解析 redis frame。如果 `self.buffer` 里的数据足够解析出一个 frame，那么这个解析出来的 frame 会从 `read_frame()` 返回。如果数据不够解析成一个 frame，我们就尝试从 socket 读取更多数据到 buffer。在读取更多数据后循环会重新开始， `parse_frame()` 会被再次调用，如此往复。这次，如果接收到了足够的数据，解析可能就会成功了。 

当从 stream 读取的时候，返回了一个 `0` 表示没有更多数据可以从对端接收了。如果这时候 read buffer 中还留有数据，这表示接收到的是个不完整的 frame 并且对端被意外中断了。这是一个错误条件，我们返回一个 `Err` 。

### The `Buf` trait

当从 stream 读取的时候， `read_buf` 被调用了。我们这个版本的 read function 带了一个参数，要求实现 [`bytes`](https://docs.rs/bytes/) crate 中的 [`BufMut`](https://docs.rs/bytes/1/bytes/trait.BufMut.html)。

首先，考虑怎样用 `read()` 实现相同的 read loop 。`Vec<u8>` 能够作为 `BytesMut` 的替代。

```rust
use tokio::net::TcpStream;

pub struct Connection {
    stream: TcpStream,
    buffer: Vec<u8>,
    cursor: usize,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream,
            // Allocate the buffer with 4kb of capacity.
            buffer: vec![0; 4096],
            cursor: 0,
        }
    }
}
```

然后是我们在 `Connection` 的 `read_frame()` 函数：

```rust
use mini_redis::{Frame, Result};

pub async fn read_frame(&mut self)
    -> Result<Option<Frame>>
{
    loop {
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // Ensure the buffer has capacity
        if self.buffer.len() == self.cursor {
            // Grow the buffer
            self.buffer.resize(self.cursor * 2, 0);
        }

        // Read into the buffer, tracking the number
        // of bytes read
        let n = self.stream.read(
            &mut self.buffer[self.cursor..]).await?;

        if 0 == n {
            if self.cursor == 0 {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        } else {
            // Update our cursor
            self.cursor += n;
        }
    }
}
```

当使用字节数组和 `read` 时，我们也必须维护一个 cursor（用来定位当前有效数据的位置）来跟踪已经有多少数据被放入了 buffer 。我们必须确保传递 buffer 的空的部分（cursor 后面的那些位置）给 `read()`，否则会覆盖掉已经塞入 buffer 的数据。如果我们的 buffer 被填满了，我们还必须为 buffer 扩容来保证可以保持读取。在 `parse_frame()` （没包含在上面），我们需要解析 `self.buffer[..self.cursor]` 中包含的数据。

因为将 byte array 和 cursor 配对是非常常见的，所以 `bytes` crate 提供了一个抽象来代表一个 byte array 和一个 cursor 。`Buf` trait 可以从能被 read 的数据实现。`Buf` trait 可以从能被 write 的数据实现。当传递一个 `T: BufMut` 给 `read_buf()` ，这个 buffer 内部的 cursor 会被 `read_buf` 自动更新。正因如此，我们这个版本的 `read_frame` 不需要管理自己的 cursor 。

此外，当使用 `Vec<u8>` 的时候，buffer 必须被**初始化**。`vec![0;4096]` 这个宏申请了一个 4k 字节的数组并且往 Vector 中的每个条目写了 0 。这个初始化过程不是免费的。当使用 `BytesMut` 和 `BufMut` 的时候，容量是**不需要**初始化的（这个特性棒:D）。`BytesMut` 这个抽象会阻止我们从未初始化的内存中进行读，这使得我们避开了初始化的步骤。

## Parsing（解析）

现在，让我们瞅瞅看 `parse_frame()` 函数。解析由两个步骤完成。

1. 确保缓冲了一个完整的 frame 并找到这个 frame 的索引位置。

2. 解析这个 frame。

`mini-redis` crate 为以上两步都提供了一个函数：

1. `Frame::check`

2. `Frame::parse`

我们还将复用 `Buf` 抽象来提供帮助。一个 `Buf` 被传递进 `Frame::check` 。当 `check` 函数迭代传进来的这个 buffer 的时候，内部的 cursor 会被推进。当 `check` 返回，这个 `Buf` 内部的 cursor 会指向 frame 的末尾。

对于 `Buf` 类型，我们会使用 [`std::io::Cursor<&[u8]>`](https://doc.rust-lang.org/stable/std/io/struct.Cursor.html) 。

```rust
use mini_redis::{Frame, Result};
use mini_redis::frame::Error::Incomplete;
use bytes::Buf;
use std::io::Cursor;

fn parse_frame(&mut self)
    -> Result<Option<Frame>>
{
    // 创建一个 `T: Buf`，Buf trait 在上面被引入了
    // self.buffer 是一个 `BytesMut`，它实现了 Deref<Target = [u8]>
    // 因此能当 [u8] 使
    let mut buf = Cursor::new(&self.buffer[..]);

    // Check whether a full frame is available
    match Frame::check(&mut buf) {
        Ok(_) => {
            // Get the byte length of the frame
            let len = buf.position() as usize;

            // Reset the internal cursor for the
            // call to `parse`.
            buf.set_position(0);

            // Parse the frame
            let frame = Frame::parse(&mut buf)?;

            // Discard the frame from the buffer
            self.buffer.advance(len);

            // Return the frame to the caller.
            Ok(Some(frame))
        }
        // Not enough data has been buffered
        Err(Incomplete) => Ok(None),
        // An error was encountered
        Err(e) => Err(e.into()),
    }
}
```

完整的 [`Frame::check`](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/frame.rs#L65-L103) 函数可以在[这里](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/frame.rs#L65-L103)找到。我们的教程不会完全覆盖到它。

需要注意的相关事项是 `Buf` 的 “byte iterator” 样式 API 被使用了。这些 API 被用来获取数据并推进内部的 cursor 。举个例子，为了操作一个 frame，首个字节被检查来决定这个 frame 的类型。这个被使用的函数是 [`Buf::get_u8`](https://docs.rs/bytes/1/bytes/buf/trait.Buf.html#method.get_u8) ，它会获取当前 cursor 的位置上的一个字节并且推 cursor 一个单位。

 [`Buf`](https://docs.rs/bytes/1/bytes/buf/trait.Buf.html) 还有很多更有用的方法。可以去 [API docs](https://docs.rs/bytes/1/bytes/buf/trait.Buf.html) 看更多细节。

## Buffered writes（带缓冲地写）

framing 的另外一半 API 是 `write_frame(frame)` 函数。这个函数会把一个完整的 frame 写入到 socket 。为了最小化 `write` 系统调用的次数，写入操作都会被缓冲(buffered)。一个 write buffer 会被维护并且在往 socket 写入之前， frame 都会被 encode 到这个 buffer。然而，不同于 `read_frame()` ，在写入 socket 之前，并不总是会缓冲一整个 frame 。

思考一下有一个批量 frame 的流 (a bulk stream frame)，被写入的值是 `Frame::Bulk(Bytes)` 。bulk frame 的报文格式是 frame 头是一个 `\$` 字符，然后跟着等同于数据字节数的长度，最后是数据本身。大部分 frame 都是 `Bytes` 的内容。如果数据很庞大，把它 copy 到一个中间缓冲区的开销会很大（这就是上一段末尾提到的）。

为了实现带缓冲的写入操作，我们将会使用 [`BufWriter` struct](https://docs.rs/tokio/1/tokio/io/struct.BufWriter.html) 。这个结构体使用 `T: AsyncWrite` 来初始化（`BufWriter::new(T)`，这个 T 得是 `AsyncWrite`），并且它本身也实现了 `AsyncWrite` 。当 `write` 在 `BufWriter` 上被调用，write 并不会直接作用到内部的 writer 上，而是作用到一个内部的 buffer 上。当这个 buffer 满了后，buffer 的内容会被刷到内部的 writer 上，同时清空这个 buffer 。我们还会有一些优化允许在某些情况下绕过缓冲区（上一段提到的情况）。

我们不会尝试把 `write_frame()` 的完整实现作为教程的一部分。所以完整实现请看[这里](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs#L159-L184)。

首先， `Connection` 结构体需要改变成如下：

```rust
use tokio::io::BufWriter;
use tokio::net::TcpStream;
use bytes::BytesMut;

pub struct Connection {
    stream: BufWriter<TcpStream>,
    buffer: BytesMut,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream: BufWriter::new(stream),
            buffer: BytesMut::with_capacity(4096),
        }
    }
}
```

接下来会实现 `write_frame()`：

```rust
use tokio::io::{self, AsyncWriteExt};
use mini_redis::Frame;

async fn write_frame(&mut self, frame: &Frame)
    -> io::Result<()>
{
    match frame {
        Frame::Simple(val) => {
            self.stream.write_u8(b'+').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Error(val) => {
            self.stream.write_u8(b'-').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Integer(val) => {
            self.stream.write_u8(b':').await?;
            self.write_decimal(*val).await?;
        }
        Frame::Null => {
            self.stream.write_all(b"$-1\r\n").await?;
        }
        Frame::Bulk(val) => {
            let len = val.len();

            self.stream.write_u8(b'$').await?;
            self.write_decimal(len as u64).await?;
            self.stream.write_all(val).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Array(_val) => unimplemented!(),
    }

    self.stream.flush().await;

    Ok(())
}
```

下面这些被用到的函数都由 [`AsyncWriteExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html) trait 提供。他们在 `TcpStream` 上也是可用的，但不建议在没有中间缓冲区的情况下发出单字节写入（一次就发一个字节，会导致太多的 syscall，太浪费资源了）。

- [`write_u8`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_u8) 把单个字节写入 writer。
- [`write_all`](https://tokio.rs/tokio/tutorial/framing) 把整个切片写入 writer。
- [`write_decimal`](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs#L225-L238) 是 mini-redis 实现的，用于把一个十进制数字转化成字符后写入。

函数以一个 `self.stream.flush().await` 调用结尾。因为 `BufWriter` 会把要写入的东西先存到一个中间缓冲区，调用 `write` 不能保证数据被写入 socket，而在返回之前我们想要 frame 被写入 socket。调用 `flush()` 会将挂在缓冲区上的所有数据写入 socket 。

另一种选择是不在 `write_frame()` 中调用 `flush()` 。相反，在 `Connection` 上提供一个 `flush()` 函数。这将允许调用者将多个小 frame 写入到缓冲区中的队列，然后使用一个 `write` syscall 将它们全部写入 socket。但是这会增加 `Connection` API 的复杂度，而简单是 Mini-Redis 的其中一个目标，所以我们决定让 `flush().await` 调用包含在 `fn write_frame()` 中。
