# Hello Tokio

我们将从编写一个非常基本的 `Tokio` 应用程序开始。它将连接到 `Mini-Redis` 服务器，设置 `key` `hello` 的值为 `world` 。然后它将读回 `key`。这将使用 `Mini-Redis` 的客户端库完成。



## 代码

---

### 生成一个新的 `crate`

```zsh
cargo new my-redis
cd my-redis
```
