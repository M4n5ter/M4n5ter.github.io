# message queue

## 消息队列杂谈

### 通过在在程序中嵌入 MQ 来缩减成本的可能性

[kafka](https://kafka.apache.org/)由 scala 和 java 编写，[rocketmq](https://rocketmq.apache.org/),[pulsar](https://pulsar.apache.org/)同样需要 java 环境，虽说它们不是不能嵌入，但是不太适合此类场景。而[rabbitmq](https://www.rabbitmq.com/)需要 erlang 环境，erlang 国内鲜有人知。activemq 已经渐渐淡出视野。

[nats](https://nats.io)也是一种消息系统，并且是[CNCF](https://landscape.cncf.io/?item=app-definition-and-development--streaming-messaging--nats)的孵化项目，不像 kafka、rocketmq 在国内大范围使用，但是同样可靠。

这一部分考虑到需要尝试一个能够被很容易嵌入的 MQ，可以尝试采用 nats，以下是一些考量：

- nats 由 go 编写，而 go 默认即是编译为单一二进制程序，并且 go 默认即是采用静态编译，所以也不需要运行的时候链接动态库，嵌入时需要考虑的面较少。
- nats 的 jetstream 提供了可靠的持久层。
- nats 内置集群能力，并且对于 jetstream cluster，使用了 nats 优化后的 raft 算法（和 2.8 以后的 kafka 的 kraft 类似）。
- nats 相对大多消息系统来说，资源占用更低，嵌入影响相对较小。



以下展示一个简短的嵌入在 go 程序中的示例，其它语言可以通过嵌入二进制 nats server 来实现。

```go
package main

import (
	"time"

	"github.com/nats-io/nats-server/v2/server"
	"github.com/nats-io/nats.go"
)

func main() {
	// 通过 server.Options 可以配置 jetstream, cluster 等选项
	server_ops := &server.Options{}

	// 通过 server_ops 初始化 server
	ns, err := server.NewServer(server_ops)
	if err != nil {
		panic(err)
	}

	// 启动 server
	go ns.Start()

	// 程序退出时正确关闭 server
	defer ns.WaitForShutdown()
	defer ns.Shutdown()

	// 等待 server 准备好接受连接
	if !ns.ReadyForConnections(5 * time.Second) {
		panic("nats server 未能在指定时间内准备好接受连接")
	}

	nc, err := nats.Connect(ns.ClientURL())
	if err != nil {
		panic(err)
	}

	subject := "demo-subject"

	// 订阅消息
	nc.Subscribe(subject, func(msg *nats.Msg) {
		println("Received message:", string(msg.Data))
	})

	// 发布消息
	nc.Publish(subject, []byte("Hello World!"))

	// 等待消息处理完成
	time.Sleep(1 * time.Second)

	// 关闭连接
	nc.Close()
}

```

```bash
# go run .
Received message: Hello World!
```



目前有个小问题就是 nats 仅支持 tcp socket，不支持 IPC,比如 unix socket。这样需要占用一个 tcp 端口，通过 localhost(127.0.0.1) 或者绑定到本机的 ip 地址进行通信虽说要经过操作系统内核网络栈，但是通常情况下至少不需要走物理网卡了。

如果介意正在使用的网络接口上的端口被占用，可以起一个虚拟接口，监听在虚拟接口上。当然，这样同样引入了一个新的接口，也许也不太舒服。

社区有人测试过，嵌入的 nats 和外部运行的 nats 在百万级的消息冲击下性能近似。

> Performance is an important aspect of every application, so let’s compare the performance for using NATS as an embedded or external service (cli, docker etc). We will run a benchmark for 1 million messages for 8 intervals.
>
> Seems like there is not much difference in performance, that’s really impressive considering we are testing for millions of messages.
>
> ![benchmark](https://raw.githubusercontent.com/m4n5ter/m4n5ter.github.io/main/assets/nats-bench.png)

