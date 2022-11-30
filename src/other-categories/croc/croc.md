# [croc](https://github.com/schollz/croc)

一个由 [schollz](https://github.com/schollz) 开发的**端到端加密**、**无需服务器或端口转发**、可传输多文件、可从意外中断中恢复传输（断点续传）、可在**任意**两台计算机直接传输文件的命令行工具。由于 `croc` 使用 GO 开发，所以天然具有跨平台功能。

schollz 开发此工具时的设计理念可在他 2019 年撰写的 [croc](https://schollz.com/blog/croc6/) 查看。

## relay > uploading

relay > uploading 是在上方那篇博客中提到的一个重要理念，`croc` 使用**中继**而不是上传的方式来进行文件传输。

我在一开始使用 `croc send <file>` 时还曾思考（当时还没有看到 croc 使用 `relay`）为何这条命令并没有携带任何关于传输给**谁**的参数，例如：

```bash
$ croc send cv_debug.log
Sending 'cv_debug.log' (282 B)   
Code is: 2151-school-biscuit-snow
On the other computer run

croc 2151-school-biscuit-snow
```

`croc` 仅仅告诉我，去另一条计算机上执行 `croc 2151-school-biscuit-snow`。在没有查看任何详细文档的情况下，以这种方式完成了一次文件传输，这让我感到非常惊讶——文件提供方没有指出要将文件发送到哪儿，文件接收方也没有提供要从哪儿接收文件，仅凭 `croc` 给出的 `2151-school-biscuit-snow` code 就得到了文件。

为了弄明白 `croc send` 的默认行为（文档比较简单，并没有给出相关说明），让我们看下 `croc` 的源码（写这篇文章时的提交是 [`cd6eb1ba53ec36a27cf8a2ac5a8b700be3e83ce3`](https://github.com/schollz/croc/commit/cd6eb1ba53ec36a27cf8a2ac5a8b700be3e83ce3)），在路径 `croc/src/cli/cli.go lines: 196~200`:

```go
    if crocOptions.RelayAddress != models.DEFAULT_RELAY {
        crocOptions.RelayAddress6 = ""
    } else if crocOptions.RelayAddress6 != models.DEFAULT_RELAY6 {
        crocOptions.RelayAddress = ""
    }
```

`models.DEFAULT_RELAY` 和 `models.DEFAULT_RELAY6` 的内容分别是 `"croc.schollz.com"` 和 `"croc6.schollz.com"` 。

从这就能看出 `relay` 并不是真正意义上的无服务器，仍然需要需要一台发送者和接收者都能发现的中继服务器，而 schollz 本人提供了默认的中继服务器。

中继服务器不需要提供真实的服务，正如其名，它仅仅只是负责中继（写到这时我并没有详细阅读 `croc` 源码，我猜测应该是像 `P2P` 一样使用类似 NAT hole punching 的技术来让端对端彼此能够发现彼此）。

`croc` 也提供了指定 `relay` 的参数，具体细节可以通过在**命令/子命令**后添加 `--help` 来查看。

## relay 是如何工作的

两端（指发送端和接收端）会连接到 relay（中继服务器），由 relay 为两端创建一个房间，该房间可容纳两个连接。连接会告诉中继服务器它需要一个房间，如果房间不存在就会创建一个新房间，如果房间已经存在并且没有满员，relay 就会将这条连接加入到它需要的房间（本情况说明这条连接是接收端，因为它想要的房间已经存在了，那个房间就是发送端向 relay 申请的房间）。

当两端都连接到 relay 时（即两端都进入了房间），relay 会为两端的连接提供一个全双工的通道，实现细节在 `croc/src/tcp lines: 388~411`:

```go
func pipe(conn1 net.Conn, conn2 net.Conn) {
    chan1 := chanFromConn(conn1)
    chan2 := chanFromConn(conn2)

    for {
        select {
            case b1 := <-chan1:
            if b1 == nil {
                return
            }
            if _, err := conn2.Write(b1); err != nil {
                log.Errorf("write error on channel 1: %v", err)
            }

            case b2 := <-chan2:
            if b2 == nil {
                return
            }
            if _, err := conn1.Write(b2); err != nil {
                log.Errorf("write error on channel 2: %v", err)
            }
        }
    }
}
```

relay 会读取一个连接中发送过来的内容，并直接将内容写入到另一个连接中。从这也能得出结论：croc 没有使用类似 nat hole punching 这样的技术，而是由中继服务器来转发传输内容，因此传输速度依旧会受到中继服务器的带宽限制。

但是当两端互相可发现（比如处于同一个局域网内），`croc` 的 `--ip value` 可以允许接收方来指定发送方的 IP 来直接从接收方获取传输内容，这样就不会经过中继服务器了。
