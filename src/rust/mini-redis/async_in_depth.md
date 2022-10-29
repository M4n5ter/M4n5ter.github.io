## Async in depth （深入异步）

至此，我们已经完成了一个相当全面的异步 Rust 和 Tokio 之旅。现在我们将会深挖 Rust 的异步运行时模型。在本教程的开始，我们就提到了 异步 Rust 用了一种独一无二的方法。现在我们来解释一下是啥意思。

## Futures

作为快速回顾，我们来举一个非常基本的异步函数。与教程到目前为止所涵盖的内容相比，这并不是什么新鲜事。

```rust
use tokio::net::TcpStream;

async fn my_async_fn() {
    println!("hello from async");
    let _socket = TcpStream::connect("127.0.0.1:3000").await.unwrap();
    println!("async TCP operation complete");
}
```

我们调用了这个函数，并且返回了某个值，对这个值调用 `.await`。

```rust
#[tokio::main]
async fn main() {
    let what_is_this = my_async_fn();
    // Nothing has been printed yet.
    what_is_this.await;
    // Text has been printed and socket has been
    // established and closed.
}
```

`my_async_fn()` 返回的值是一个 future ，future 是一个实现了标准库提供的  [`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 的值。它们是包含正在进行的异步计算的值。

 [`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 的定义如下：

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Self::Output>;
}
```

关联类型( [associated type](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types) ) `Output` 是 future 一旦完成后会产生的类型。可以通过看标准库文档（[standard library](https://doc.rust-lang.org/std/pin/index.html)）得到更多细节。

不像其它语言实现的 future ，一个 Rust 的 future 不是代表一个正在后台发生的计算，而是 Rust future 就是计算本身。future 的所有者负责通过 poll the future 来推动计算，这就是 `Future::poll` 所做的事。

### Implementing `Future` （实现 `Future`）

让我们实现一个简单的 future。这个 future 将会：

1. 一直 wait 到特定时刻。

2. 输出一些文本到 STDOUT 。

3. 产生一个字符串。

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
}
```

### Async fn as a Future （异步函数作为 future）

在 main 函数中，我们实例化一个 future 并对它调用 `.await` 。在异步函数中，我们可以对任何实现了 `Future` 的值调用 `.await` 。相反，调用一个 `async` function 返回一个实现了 `Future` 的匿名类型。`async fn main()` 所生成的 future 类似于：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
    // Initialized, never polled
    State0,
    // Waiting on `Delay`, i.e. the `future.await` line.
    State1(Delay),
    // The future has completed.
    Terminated,
}

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use MainFuture::*;

        loop {
            match *self {
                State0 => {
                    let when = Instant::now() +
                        Duration::from_millis(10);
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => {
                    match Pin::new(my_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            *self = Terminated;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```

Rust 的 future 是**状态机**(`state machine`) 。此处，`MainFuture` 代表着由一个 future 可能的状态构成的 `enum` 。这个 future 从 `State0` 状态开始，当 `poll` 被调用时，这个 future 会尽可能地尝试推动其内部的状态。如果这个 future 能够完成了，`Poll::Ready` 会返回它包含的异步计算的输出结果。

如果这个 future **不**能够完成，通常是由于资源问题，这种情况它一般还在等着被调度，等着变成 `Poll::Ready` ，这时会返回 `Poll::Pending` 表示 future 还没完成。收到 `Poll::Pending` 表示告诉 future 的调用者，这个 future 将会在之后一段时间被完成，并且调用者应该在之后再次调用 `poll` 。

我们也看到了 future 由 其他 future 构成（future 可以嵌套）。对外层的 future 调用 `poll` 会导致内部的 future 的 `poll` 函数也被调用。  

## Executor （执行者，一般就是运行时了）

异步 Rust 函数会返回 future ，而 future 又必须通过调用它们身上的 `poll` 来推进它们的状态，future 又由其它 future 组成。因此，问题来了，谁来调用最最最外层的 future 的 `poll` 呢？

回顾之前的内容，为了运行异步函数，它们也必须被传递给 `tokio::spawn` 或者 main 函数被用 `#[tokio::main]` 注释。这都会把生成的外层 future 提交给 Tokio executor ，这个 executor 负责调用外层 future 的 `Future::poll` 来驱动异步计算完成。

### Mini Tokio

为了更好地理解这一切是如何结合在一起的，让我们实现我们自己的 minimal version Tokio！ 完整代码能在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/mini-tokio/src/main.rs)被找到。

```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

fn main() {
    let mut mini_tokio = MiniTokio::new();

    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };

        let out = future.await;
        assert_eq!(out, "done");
    });

    mini_tokio.run();
}

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }

    /// Spawn a future onto the mini-tokio instance.
    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }

    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker);

        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

运行了一个 async block，使用自定义的 delay  创建了一个 `Delay` future 实例并且调用了 `.await` 。然而，我们的目前为止的实现有一个重大的**污点**，那就是我们的执行者永远不会 sleep，执行者在持续不断的循环所有生成的 future 并且 poll 它们。大多数时候，future 们都没有准备好执行更多的工作并且会再次返回 `Poll:Pending` （所以应该需要有一定的间隔，而不是没有 sleep 的无限循环去 poll）。这个过程会大量消耗 CPU 资源并且通常并不高效。

理想情况下，我们希望 mini-tokio 只在 future 能够取得进展时才进行 poll 。这种情况会发生在当任务被阻塞时的资源准备好去执行被请求的操作的时候。如果任务想要从一个 TCP socket 读取数据，那么我们只希望当 TCP socket 已经接收到数据的时候才去 poll 任务（而不是 socket 里啥都没有的时候去疯狂 poll） 。在我们的场景下，任务被阻塞直到给出的 `Istant` 到达，理想情况下，mini-tokio 应该只在那一时刻刚过后去 poll 任务。

为了实现这个目的，当一个资源被 poll，并且这个资源**没有**准备好时，这个资源将会在它转变成 ready state 的时候主动发送一个通知。

## Wakers （唤醒者）

Waker 是缺失的部分，这是资源能够通知正在等待的任务资源已准备好继续某些操作的一个系统（换句话说就是 waker 负责通知外面等我的那个任务，告诉它我准备好了，来 poll 我吧）。

让我们再看看 `Future::poll` 的定义：

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context)
    -> Poll<Self::Output>;
```

可以发现想要 poll future 的时候需要携带一个 [`Context`](https://doc.rust-lang.org/std/task/struct.Context.html) ，而它有一个 `waker()` 方法，这个方法返回一个 [`Waker`](https://doc.rust-lang.org/std/task/struct.Waker.html) 绑定到当前任务。这个 [`Waker`](https://doc.rust-lang.org/std/task/struct.Waker.html) 有一个 `wake()` 方法，这个方法正是我们要的，调用这个方法会发送信号给 executor，表示相关联的任务应该被调度来执行了。当资源转变成 ready state 的时候调用 `wake()` 方法来通知 executor 可以 poll 任务来获取进展。

### Updating `Delay`

我们可以更新 `Delay` 来使用 wakers ：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use std::thread;

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Get a handle to the waker for the current task
            let waker = cx.waker().clone();
            let when = self.when;

            // Spawn a timer thread.
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                waker.wake();
            });

            Poll::Pending
        }
    }
}
```

现在，一旦指定的时间到了，调用的任务会通知 executor 并且 executor 能够确保任务再次被调度。下一步就是更新 mini-tokio 来监听 wake notifications（通知）。

这里我们的 `Delay` 实现仍然留有一些问题。我们将会在后面修复它们。

> 当一个 future 返回 `Poll::Pending` ，它**必须**确保 waker 是在某一点被注册了。忘记这么做会导致任务被无限期地挂起（因为没 waker 去通知 executor 来 poll 了）。
> 
> 忘记在返回 `Poll::Pending` 后 wake 一个 task 是一个常见的 bug 来源。

回看一下 `Delay` 的第一次迭代。这是 future 的实现：

```rust
impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

当时被注释暂时先忽略的那一行：咱返回 `Poll::Pending` 前，我们调用了 `cx.waker().wake_by_ref()` 。这是为了满足 future 的约定。通过返回 `Poll::Pending` 我们负责向 waker 发送 wake 信号。。因为我们暂时还没有实现 timer thread，所以我们直接用 inline 的方式向 waker 发送了信号。这么做会导致这个 future 被立即再调度(re-scheduled)，再次执行，并且可能还是没有转变成 ready state 。

请注意，你可以更频繁的向 waker 发送信号，而不必是必须必要的时候才发送信号。在这种特殊情况下，我们向 waker 发送信号，即使我们根本还没准备好继续操作。除了会浪费一些 CPU 资源外没有任何不对的。然而这种特殊的实现将会导致一个 busy loop 。

### Updating Mini Tokio

接下来就是改变我们的 Mini Tokio 来接收 waker notifications 。我们希望 executor 只在它们被唤醒的时候执行任务，为了做到这点， Mini tokio 将会提供它自己的 waker 。当这个 waker 被调用，它所关联的任务就会排队来执行。Mini-Tokio 在 poll future 的时候会把它的 waker 传递给 future。

更新后的 Mini Tokio 将会使用一个 channel 来存储被调度的任务。channel 允许从任何线程来排队执行任务。Wakers 必须是实现了 `Send` 和 `Sync` 的，因此我们可以使用来此 [`crossbeam`](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html) crate 的 channel，因为标准库的 channel 没实现 `Sync` 。

> `Send` 和 `Sync` traits 是 Rust 提供的关于并发的“标记 trait“。能被 **send** 到不同线程的类型是 `Send` 。大多数类型都是 `Send` ，但是有些像 [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html) 这样的不是。类型能被通过不可变引用被**并发**访问的是 `Sync` 。一个类型可以是 `Send` 但不一定是 `Sync` — 一个很好的例子就是 [`Cell`](https://doc.rust-lang.org/std/cell/struct.Cell.html) ，可以通过不可变引用来修改内容（内部可变性），因此通过并发访问是不安全的。
> 
> 更多细节可以看  [the Rust book 中相关的章节](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html) 。

把下面的依赖加到 `Cargo.toml` 来获取我们需要的 channel 。

```toml
crossbeam = "0.8"
```

然后改 `MiniTokio` 结构体。

```rust
use crossbeam::channel;
use std::sync::Arc;

struct MiniTokio {
    scheduled: channel::Receiver<Arc<Task>>,
    sender: channel::Sender<Arc<Task>>,
}

struct Task {
    // This will be filled in soon.
}
```

Wakers 是 `Sync` 并且可以被 clone。当 `wake` 被调用，任务必须被调度来执行。为了实现这个目的，我们整了个 channel 。当 `wake()` 在 waker 身上被调用时，任务会被推进 channel 的 send 的那一半（channel 被拆成两半，一半 send 一半 receive）。我们的 `Task` 结构体将会实现 wake 逻辑。为了做到这点，它需要同时包含生成的任务和 channel 的 send 。

```rust
use std::sync::{Arc, Mutex};

struct Task {
    // The `Mutex` is to make `Task` implement `Sync`. Only
    // one thread accesses `future` at any given time. The
    // `Mutex` is not required for correctness. Real Tokio
    // does not use a mutex here, but real Tokio has
    // more lines of code than can fit in a single tutorial
    // page.
    future: Mutex<Pin<Box<dyn Future<Output = ()> + Send>>>,
    executor: channel::Sender<Arc<Task>>,
}

impl Task {
    fn schedule(self: &Arc<Self>) {
        self.executor.send(self.clone());
    }
}
```

为了调度任务，`Arc` 会被 clone 然后通过 channel 发送出去。现在，我们需要将我们的 `schedule` 函数和 [`std::task::Waker`](https://doc.rust-lang.org/std/task/struct.Waker.html) 挂钩。标注版酷提供了一个低层次 API ，通过 [manual vtable construction](https://doc.rust-lang.org/std/task/struct.RawWakerVTable.html) （手动构造 vtable，vtable 能够产生晚绑定行为，只有在运行时才知道调用的是什么函数，例如调用 vtable 中的 A，然后会把 A 映射的函数指针 *B 拿出来执行）来做这件事。这个方案为实现者提供了最大程度的灵活性，但是要求一大堆 unsafe 样板代码。与直接使用 [`RawWakerVTable`](https://doc.rust-lang.org/std/task/struct.RawWakerVTable.html) 相反，我们将会使用  [`futures`](https://docs.rs/futures/) crate 提供的 [`ArcWake`](https://docs.rs/futures/0.3/futures/task/trait.ArcWake.html) trait，它允许我们通过实现一个简单的 trait 来暴露我们的 `Task` 结构体作为一个 waker 。

把下面的依赖加入到 `Cargo.toml`。

```toml
futures = "0.3"
```

然后实现  [`futures::task::ArcWake`](https://docs.rs/futures/0.3/futures/task/trait.ArcWake.html) 。

```rust
use futures::task::{self, ArcWake};
use std::sync::Arc;
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.schedule();
    }
}
```

当之前的那个 timer thread 调用 `waker.wake()` ，任务会被推进 channel 。接着我们实现一下 `MiniTokio::run()` 函数中的接收并执行任务的部分。

```rust
impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    /// 初始化 mini-tokio 实例
    fn new() -> MiniTokio {
        let (sender, scheduled) = channel::unbounded();

        MiniTokio { scheduled, sender }
    }

    /// 生成一个 future 加到 mini-tokio 实例上
    /// 
    /// 把接收到的 future 包装进 `Task` ，`Task` 可以把自己发送到
    /// `scheduled` queue。然后里面包装的 future 就能在 mini-redis 实例的
    /// `run` 调用中被拿出来执行了。
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        Task::spawn(future, &self.sender);
    }
}

impl Task {
    fn poll(self: Arc<Self>) {
        // 从 `Task` 创建一个 waker。这使用了上面我们给 `Task`
        // 实现的 `ArcWake` trait，这个 `waker` 方法就是 `ArcWake` 的，
        // 用来从实现了 `ArcWake` trait 的类型上生成一个 waker 
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);

        // No other thread ever tries to lock the future
        let mut future = self.future.try_lock().unwrap();

        // Poll the future
        let _ = future.as_mut().poll(&mut cx);
    }

    // Spawns a new task with the given future.
    //
    // Initializes a new Task harness containing the given future and pushes it
    // onto `sender`. The receiver half of the channel will get the task and
    // execute it.
    fn spawn<F>(future: F, sender: &channel::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            future: Mutex::new(Box::pin(future)),
            executor: sender.clone(),
        });

        let _ = sender.send(task);
    }

}
```

这里发生了很多事情。首先，`MiniTokio::run()` 被实现了，这个函数启动了一个 loop 从 channel 接收被调度的任务。因为任务在被唤醒的时候会被推进这个 channel，所以这些任务能够在要执行的时候顺利取得进展。

此外，`MiniTokio::new()` 和 `MiniTokio::spawn()` 函数被调整为使用一个 channel 而不是一个 `VecDeque` 。当新的任务产生，它们会被给到这个 channel 的 send 的 clone，使得任务可以在运行时调度自己（通过把自己塞进 send 里送到 channel 中去）。

`Task::poll()` 函数会通过手动为  `Task` 实现的 `future` crate 中的  [`ArcWake`](https://docs.rs/futures/0.3/futures/task/trait.ArcWake.html) trait 来创建 waker 。这个 waker 被用来创建一个 `task::Context` ，然后这个 `task::Context` 被传给 `poll` 。

## Summary （概括）

我们现在已经看到了异步 Rust 如何工作的端到端示例。Rust 的 `async/await` 特性由 traits  支持。这就允许了第三方 crates，像 Tokio，来提供执行细节。

* 异步 Rust 的操作是惰性的，并且需要一个调用者去 poll 它们。

* Waker 会被传递给 futures 来把一个 future 和调用它的任务联系起来。

* 当一个资源**没有**准备好完成一个操作时，会返回 `Poll::Pending` 并且任务的 waker 会记录这点。

* 当资源 ready 时，任务的 waker 会发送通知。

* executor 接收到通知并且调度任务去执行。

* 当任务再次被 poll 的时候，此时资源已经就绪了，并且任务会取得进展。  

## A few loose ends （一些零散的内容放在结尾）

回想一下，当我们之前在实现 `Delay` 这个 future 的时候，我们说过还有一些事情需要解决。Rust 的异步模型允许单个 future 在多个任务之前迁移。思考下下面的内容：

```rust
use futures::future::poll_fn;
use std::future::Future;
use std::pin::Pin;

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let mut delay = Some(Delay { when });

    poll_fn(move |cx| {
        let mut delay = delay.take().unwrap();
        let res = Pin::new(&mut delay).poll(cx);
        assert!(res.is_pending());
        tokio::spawn(async move {
            delay.await;
        });

        Poll::Ready(())
    }).await;
}
```

`poll_fn` 函数使用闭包创建了一个 `Future` 实例，上面的片段中创建一个 `Delay` 实例，poll 了一下它，然后把 `Delay` 实例发送到了一个新的任务中去进行 `.await` 。在这个例子里， `Delay::poll` 被**不同**的 `Waker` 调用了超过一次。当发生这种情况，你必须确保在 传递给了*最近的* 那次 `poll` 的 `Waker` 上的 `wake` 被调用。

当实现一个 future 的时候，假设每次对 `poll` 的调用**可能**被应用到一个不同的 `Waker`  实例是非常重要的。poll 函数必须更新任何先前记录的 waker 为最新传给它的 waker 。

我们先前实现的 `Delay` 在每次被 poll 的时候都会生成一个新的线程。这当然也 ok，但是如果它被 poll 的太频繁的话就会变得非常低效。（e.g. 如果你对这个 future 和其它 future 使用了 `select!` ，那么不论他俩哪个发生了事件，两者都会被调用）。一种方法是记住你是否已经创建过一个线程，并且只在你没有创建过时去生成一个新线程。然而，如果你这么做了，你必须确保线程的 `Waker` 被更新为最近的一次 poll 的 `Waker` ，因为你不这么做的话就无法唤醒最近的那个 `Waker` 。

为了修复前面的那个实现，我们可以像这样做：

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
    // 当我们已经生成了一个线程时这里是 Some，否则是 None。
    waker: Option<Arc<Mutex<Waker>>>,
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // 首先，如果这是 future 第一次被调用，那就生成一个 timer thread。
        // 如果 timer thread 已经在运行了，确保存储的 `Waker` 匹配当前任务的 waker。
        if let Some(waker) = &self.waker {
            let mut waker = waker.lock().unwrap();

            // 检查存储的 waker 是否匹配当前任务的 waker。
            // 当 `Delay` future 在可能会被移动到不同的任务 `poll` 时，
            // 这是很有必要的。如果这种情况发生了，`Context` 中包含的 waker 会不一样
            // 并且我们必须更新我们在 `Delay` 存储的 waker 来应对变化。
            if !waker.will_wake(cx.waker()) {
                *waker = cx.waker().clone();
            }
        } else {
            let when = self.when;
            let waker = Arc::new(Mutex::new(cx.waker().clone()));
            self.waker = Some(waker.clone());

            // 这是第一次调用 `poll` 的情况，生成一个 timer thread。
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                // The duration has elapsed. Notify the caller by invoking
                // the waker.
                let waker = waker.lock().unwrap();
                waker.wake_by_ref();
            });
        }

        // 一旦 waker 被存储了，并且 timer thread 开始了，就到了检查
        // delay 是否完成的时候了。这通过检查当前的 instant 来做到。
        // 如果时间到了，那么就意味着 future 已经完成，并且得返回 `Poll::Ready`
        if Instant::now() >= self.when {
            Poll::Ready(())
        } else {
            // 时间还没到，future 还没完成，返回 `Poll::Pending`。
            //
            // `Future` trait 约定了：当 `Pending` 被返回时，future 确保
            // 一旦再次被 poll 就会往给定的 waker 发送信号。在我们的情况下，
            // 通过在这返回 `Pending`，我们承诺一旦请求的时间过了，我们将会调用包含在 `Context`
            // 参数内的 waker。我们通过在上面生成一个 timer thread 来确保这点。
            //
            // 如果我们忘记调用 waker，这个任务将会无期限的挂起。
            Poll::Pending
        }
    }
}
```

这有一点复杂，但是我们的想法是，对每个 `poll` 的调用，future 都会检查当前 `poll` 给的 waker 跟之前记录的 waker 是不是匹配的。如果两个 waker 匹配，那么就不会做其它的事情了。如果它们不匹配，那么就记录最新的 `poll` 里的 waker。

### `Notify` utility

我们演示了如果使用 waker 来手动实现 `Delay` future。Wakers 是异步 Rust 如何去工作的基础。通常，没有必要去降低到那样的 level（手动实现 future 是一种偏向底层的行为）。举个例子，在这个 `Delay` 的场景，我们可以通过使用 [`tokio::sync::Notify`](https://docs.rs/tokio/1/tokio/sync/struct.Notify.html) 实用工具纯使用 `async/await` 来实现它。这个实用工具提供了一个基本的任务通知机制，它会处理 waker 的细节，包括确保记录的 waker 匹配当前的 task 。

使用 [`Notify`](https://docs.rs/tokio/1/tokio/sync/struct.Notify.html) ，我们可以像这样使用 `await` 实现一个 `delay` 函数：

```rust
use tokio::sync::Notify;
use std::sync::Arc;
use std::time::{Duration, Instant};
use std::thread;

async fn delay(dur: Duration) {
    let when = Instant::now() + dur;
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    thread::spawn(move || {
        let now = Instant::now();

        if now < when {
            thread::sleep(when - now);
        }

        notify2.notify_one();
    });


    notify.notified().await;
}
```
