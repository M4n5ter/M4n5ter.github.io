# [croc](https://github.com/schollz/croc)

一个由 [schollz](https://github.com/schollz) 开发的**端到端加密**、**无需服务器或端口转发**、可传输多文件、可从意外中断中恢复传输（断点续传）、可在**任意**两台计算机直接传输文件的命令行工具。由于 `croc` 使用 GO 开发，所以天然具有跨平台功能。

schollz 开发此工具时的设计理念可在他 2019 年撰写的 [croc](https://schollz.com/blog/croc6/) 查看。

## relay > uploading

relay > uploading 是在上方那篇博客中提到的一个重要理念，`croc` 使用**中继**而不是上传的方式来进行文件传输。

我在一开始使用 `croc send <file>` 时还曾思考（当时还没有看到 croc 使用 `relay`）为何这条命令并没有携带任何关于传输给**谁**的参数，例如：

```bash
\$ croc send cv_debug.log
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

中继服务器不需要提供真实的服务，正如其名，它仅仅只是负责中继（写这篇文章时我并没有详细阅读 `croc` 源码，我猜测应该是像 `P2P` 一样使用类似 DNS hole-punch 的技术来让端对端彼此能够发现彼此）。

`croc` 也提供了指定 `relay` 的参数，具体细节可以通过在**命令/子命令**后添加 `--help` 来查看。

## 写在最后

`croc` 的设计理念非常棒，让我学到了文件传输的新姿势，这种设计我觉得完全可以应用在许多地方，例如在 IM System 中引入加密传输（不在服务器上存储任何数据，除了发送者和接收者外没有人能够知道传输内容），感兴趣的可以阅读开头的那篇博客。
