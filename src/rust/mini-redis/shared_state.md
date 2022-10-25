## Shared state

到目前为止，我们有一个 key-value server 在工作。但是，存在一个重大缺陷：状态不会在连接之间共享。我们将在本文中修复它。

## Strategies （方案）

这里有两种不同的方式来在 Tokio 中分享状态。

1. 使用 `Mutex` 保护被分享的状态。

2. 生成一个任务来管理状态并且使用**消息传递**来操作

一般来说你想要为简单的数据采用第一种方法，第二种方法用来应对需要像 I/O 原语这样的异步工作。在本章，被分享的状态是一个 `HashMap` 并且操作为 `insert` 和 `get` 。这两种操作都不是异步的，因此我们可以使用 `Mutex` 。

下一章再来介绍后一种方法。

## Add `bytes` dependency （添加 `bytes` 依赖）

与使用 `Vec<u8>` 不同，Mini-Redis crate 使用了 [`bytes`](https://docs.rs/bytes/1/bytes/struct.Bytes.html) crate 中的 `Bytes` 。使用 `Bytes` 的目的是为网络编程提供一个健壮的字节数组结构体。`Bytes` 比 `Vec<u8>` 多的一个最大的特点是它实现了浅拷贝。换句话说，在 `Bytes` 实例上调用 `clone()` 不会拷贝底层的数据。相反，一个 `Bytes` 实例是一个底层数据的 rc（引用计数器）句柄。`Bytes` 类型与 `Arc<Vec<u8>>` 相似，但是多了些附加的功能。

为了引入 `bytes` 依赖，把下方的内容添加到你的 `Cargo.toml` 中的 `[dependencies]` 部分：

```toml
bytes = "1"
```

## Initialize the `HashMap` （初始化 `HashMap`）

`HashMap` 将会被跨多任务（并且可能会是多个线程）共享。为了能够做到这点，它将会被 `Arc<Mutex<_>>` 包裹。

首先，方便起见，在 `use` 语句后加上下面的类型别名：

```rust
use bytes::Bytes;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

type Db = Arc<Mutex<HashMap<String, Bytes>>>;
```

然后，改变 `main` 函数来初始化 `HashMap` 并且传递一个 `Arc` 句柄参数给 `process` 函数。使用 `Arc` 能够允许 `HashMap` 被多个任务并发地引用以及在多线程中运行。在整个 Tokio 中，这样的 `Arc` 句柄常见于用来引用一个提供了对某些共享状态的访问的值。

```rust
use tokio::net::TcpListener;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    println!("Listening");

    let db = Arc::new(Mutex::new(HashMap::new()));

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // Clone the handle to the hash map.
        let db = db.clone();

        println!("Accepted");
        tokio::spawn(async move {
            process(socket, db).await;
        });
    }
}
```

### 使用 `std::sync::Mutex`

请注意，用的是 `std::sync::Mutex` 来保护 `HashMap` 而不是 `tokio::sync:Mutex` 。一个常见的错误是无条件的在异步代码中使用 `tokio::sync::Mutex` 。异步锁是用来锁定跨 `.await` 调用的互斥锁。

一个同步的互斥锁在等待获取锁的时候会阻塞当前线程。所以反过来说，它会阻塞所在线程对其它任务的处理。但是，切换到 `tokio::sync::Mutex` 通常不能够有什么帮助，因为异步锁在内部也使用了同步锁。

有这样一个经验法则，只要锁竞争保持在一个较低的水准并且锁没有跨 `.await` 持有，那么在异步代码中使用同步锁也很好。另外，可以考虑使用 [`parking_log::Mutex`](https://docs.rs/parking_lot/0.10.2/parking_lot/type.Mutex.html) 作为替代，它是比 `std::sync::Mutex` 更快的实现。

## Update `process()` （更新 `process()`）

这个函数不再初始化 `HashMap` 。相反，它接收一个共享的 `HashMap` 作为参数。它同样需要在使用前 lock 这个 `HashMap` 。请记住，HashMap 的值的类型现在是 `Bytes` （clone 它的代价非常低）了，所以也需要修改。

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream, db: Db) {
    use mini_redis::Command::{self, Get, Set};

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                let mut db = db.lock().unwrap();
                db.insert(cmd.key().to_string(), cmd.value().clone());
                Frame::Simple("OK".to_string())
            }           
            Get(cmd) => {
                let db = db.lock().unwrap();
                if let Some(value) = db.get(cmd.key()) {
                    Frame::Bulk(value.clone())
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

### Tasks, threads, and contention （任务、线程、竞争）

当锁竞争很小的时候，使用一个阻塞的锁来保护`短临界区` 是一种可接受的策略。当一个锁在被竞争，执行本任务的线程必须阻塞并且等待这个锁。这不仅仅会阻塞当前的任务，还会阻塞其他被调度到当前线程上的任务。

默认情况下，Tokio runtime 使用一个多线程调度器。任务被调度到被 runtime 管理的任意数量的线程上。如果计划执行大量任务，并且它们都需要访问互斥锁，那么就会出现竞争。另一方面，如果 [`current_thread`](https://docs.rs/tokio/1.21.2/tokio/runtime/index.html#current-thread-scheduler) runtime 风格被启用，那么互斥锁将永远不会被竞争。

>  [`current_thread` runtime flavor](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_current_thread) 是一个轻量、单线程的运行时。当只生成少量任务和打开不多的 sockets 时它是一个不错的选择。举个例子，当在异步客户端库之上桥接一个同步 API 时，这种选择效果很好（比如用new 一个 current_thread runtime，然后在它之上用 block_on 执行异步代码）。



如果在同步锁上的竞争成为了一个问题，最好的解决方案是少量切换成 Tokio mutex。如果不采用前者方案，要考虑的选项有：

* 跑一个专门用来管理状态的任务，并且使用消息传递来共享状态。

* 分片锁。

* 重构代码来避开锁。

在我们目前的情况下，因为每个 key 都是独立的，所以分片锁的效果会很棒！为了做到这个，不能够只有一个单独的 `Mutex<HashMap<_,_>>` 实例，我们需要引入 `N` 个不同的实例：

```rust
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;

fn new_sharded_db(num_shards: usize) -> ShardedDb {
    let mut db = Vec::with_capacity(num_shards);
    for _ in 0..num_shards {
        db.push(Mutex::new(HashMap::new()));
    }
    Arc::new(db)
}
```

接着，找到给定 key 的的位置变成了两步过程。第一步，用 key 来确定在哪一个hash map 分片。第二步在 `HashMap` 中找 key：

```rust
let shard = db[hash(key) % db.len()].lock().unwrap();
shard.insert(key, value);
```

上面概述的简单实现需要使用固定数量的分片，并且一旦创建了 `SharedDb` 后分片的数量就不能改变了。[dashmap](https://docs.rs/dashmap) crate 提供了一个更有经验验证的分片 hash map 实现。



## Holding a `MutexGuard` across an `.await` （跨 `.await` 持有一个 `MutexGuard`）

你可能会写出像下面这样的代码：

```rust
use std::sync::{Mutex, MutexGuard};

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;

    do_something_async().await;
} // 锁在这里超出作用域

```

当你尝试 spawn 一些东西来调用这个函数，你会遇到下面的错误信息：

```rust
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```

这个错误会发生是因为 `std::sync::MutexGuard` 类型没有实现 `Send` trait 。这意味着你不能传递一个同步锁到另一个线程，另一个原因是 Tokio runtime 在每个 `.await` 调用时能够在线程间 move 一个任务。为了避免这个错误，你应该重构你的代码来让互斥锁的析构函数在 `.await` 之前就运行完毕。

```rust
// 这样就行了！
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    {
        let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
        *lock += 1;
    } // 锁在这里超出作用域

    do_something_async().await;
}
```

值得注意的是，下面这样不能正常运作：

```rust
use std::sync::{Mutex, MutexGuard};

// This fails too.
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    drop(lock);

    do_something_async().await;
}
```

这是因为编译器目前只是通过作用域信息来计算一个 future 是不是实现了 `Send` trait 。编译器未来有望被更新来支持显式的 drop，但是现在咱只能显式的加上作用范围。

注意，这里讨论的错误在上一节的 [Send Bound - Spawning](https://m4n5ter.github.io/rust/mini-redis/spawning.html#send-bound) 也讨论过。

你不应该尝试通过某种方式生成一个不需要实现 `Send` 的任务来规避这个问题，因为
