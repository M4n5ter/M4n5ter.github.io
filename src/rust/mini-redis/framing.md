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



## Buffered reads （带缓冲的读）

`read_frame` 方法在返回前会等待一个完整的 frame 被接收。
