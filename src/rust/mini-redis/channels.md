## Channels TODO

现在我们已经了解了一些关于 Tokio 的并发，让我们把它们应用到客户端侧吧。把我们先前写的服务端的代码移动到一个显式的二进制文件里去：

```zsh
mkdir src/bin
mv src/main.rs src/bin/server.rs
```
