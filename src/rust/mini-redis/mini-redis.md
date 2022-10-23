## 介绍

[mini-redis](https://github.com/tokio-rs/mini-redis) 是一个不完整的使用 [tokio](https://github.com/tokio-rs/tokio) 构建的 [redis](https://redis.io/) client 和 server 。

是 tokio 团队提供的一个用于学习 `tokio` 的稍大的示例项目，接下来将以 [mini-redis](https://github.com/tokio-rs/mini-redis) 作为我的第一个用来学习 `Rust` 的项目。

该项目的目的即是进行 `tokio` 教学([Tokio Tutorial](https://tokio.rs/tokio/tutorial))，所以接下来就跟着 [Tokio Tutorial](https://tokio.rs/tokio/tutorial) 走吧～

## 开始

`Tokio Tutorial` 将会带着我们逐步完成 `mini-redis` 的客户端和服务端。从使用 Rust 进行异步编程的基础知识开始，并从那里开始构建。我们将实现Redis命令的一个子集，但将全面了解Tokio。

## 获得帮助

 `tokio` 的 [Discord](https://discord.gg/tokio) 和 [GitHub discussions](https://github.com/tokio-rs/tokio/discussions) 是初学者获得帮助的好地方，在那里不用担心提一些“初学者才会提的问题”，大家都是从某个地方开始，很乐意帮忙。

## 先决条件

在该教程中说明了教程需要读者已经熟悉了 `Rust` 编程语言，并且推荐了 [Rust book](https://doc.rust-lang.org/book/)，当然 rust cn 社区有一本同样优秀的 [Rust course](https://course.rs) 。

虽然不是必需的，但有使用Rust标准库或其他语言编写网络代码的一些经验可能会有所帮助。

### Rust

本教程至少需要Rust版本1.45.0，但建议使用Rust的最新稳定版本。

```zsh
rustc --version
rustc 1.64.0 (a55dd71d5 2022-09-19)
```

### Mini-Redis server

接下来需要安装 `Mini-Redis server` 来保证我们写的客户端能被测试。

```zsh
cargo install mini-redis
```

如果因为国内糟糕的网络环境导致下载速度不忍直视，可以使用字节跳动 Rust 技术团队的 [rsproxy](https://rsproxy.cn/) 来替换默认源。

通过启动 server 来确保我们已经成功安装。

```zsh
mini-redis-server
```

接着另外打开一个终端窗口，尝试使用`mini-redis-cli` `get` 一个 `key` 。

```zsh
mini-redis-cli get foo
```

不出意外你会看到 `(nil)` 。

## 准备开始

就是这样，一切准备就绪。转到下一页编写我们的第一个异步Rust应用程序。
