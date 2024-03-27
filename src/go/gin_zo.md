## ZincObserve

[ZincObserve](https://github.com/zinclabs/zincobserve) （以下简称 ZO）是由 zinclabs 开源的一款日志搜索引擎（他们之前有一款产品叫做 zincsearch，现在已经基本由社区驱动，原因是他们最初的目的是开发一款在日志搜索领域做的最好的软件，现在已经全部精力投入到 ZincObserve了），目的是在 logs,metrics,traces 搜索领域替代 elasticsearch ，我们都知道 elasticsearch 非常笨重，资源消耗极大，而 ZO 专精与日志搜索，所以全文搜索、向量搜索等请移步 meilisearch、zincsearch、elasticsearch、qdrant 等其它软件。

ZO 使用 Rust 开发（他们家的 zincsearch 是用的 go），性能极高，**存储成本仅仅为 elasticsearch 的 1/140**。目前 ZO 的后端部分已经基本稳定，现在提出的 bugs 基本都是前端问题。虽然 ZO 从版本号来看还没有为生产做好准备，但是鉴于 zincsearch 的优良表现，我们完全可以相信 zinclabs 的能力（毕竟他们创立的目的就是为了开发一款世界上最好的日志搜索软件）。

## 将 Gin 的日志输出到 ZincObserve

### zincobserve

ZO 支持 win/linux/mac/docker/k8s ，这里我直接从 [Releases · zinclabs/zincobserve · GitHub](https://github.com/zinclabs/zincobserve/releases) 下载

```bash
wget https://github.com/zinclabs/zincobserve/releases/download/v0.4.1/zincobserve-v0.4.1-linux-amd64.tar.gz
tar zxvf zincobserve-v0.4.1-linux-amd64.tar.gz
```

ZO 是通过环境变量的方式来配置的，会从 `.env` 读取，这里就指定一下最小配置量的环境变量。

```bash
$ cat .env
ZO_ROOT_USER_EMAIL=admin@m4n5ter.email
ZO_ROOT_USER_PASSWORD=123456
$ ./zincobserve
```

### fluent-bit

首先，我们需要有东西来采集日志，这里我使用 [fluent-bit](https://docs.fluentbit.io/) ，fluent-bit 支持大量的 input 和 output 插件，其中 output 插件有 elasticsearch 支持，这里提到这个插件是因为 **ZO 兼容 ES API，可以直接使用 ES output plugin**，但是我这里不使用这个插件，而是直接使用 http output plugin。

而 input 插件我这里就使用 tcp input ，   其实 tail 插件也可以，tail 支持从文件系统中读取日志，只需要将 gin 的日志输出一份到磁盘就行。但是使用 tcp input 的话，我们可以将fluent-bit 放在其它地方，只要网络可达就能采集日志，更加灵活。

首先下载一个 fluent-bit，这里使用 docker 的方式，方便一些：

```bash
docker pull cr.fluentbit.io/fluent/fluent-bit:2.1.1
```

具体 tags 可以去 fluent-bit 官网那看看需要哪个版本即可。

准备一份配置文件:

config:

```unit
[INPUT]
    Name        tcp
    Listen      0.0.0.0
    Port        5170
    Chunk_Size  32
    Buffer_Size 64
    Format      json

[OUTPUT]
  Name http
  Match *
  URI /api/default/test/_json
  Host localhost
  Port 5080
  tls Off
  Format json
  Json_date_key    _timestamp
  Json_date_format iso8601
  HTTP_User admin@m4n5ter.email
  HTTP_Passwd uoZ9nMUEywjSLAiP
```

这里的 [OUTPUT] 直接访问 <http://>`<IP>`:`<Port>` 在 ZO 的 WEB 界面选择采集（ingestion）后选择 fluent-bit 就能得到。`URL /api/{组织}/{数据流}/_json` ，这里的组织和数据流随便都行， 没有的话 ZO 会自动创建。

```bash
$ docker run -it --network host -v .:/data --rm --name fluent-bit cr.fluentbit.io/fluent/fluent-bit:2.1.1 \
-c /data/config
```

直接前台运行 fluent-bit 方便直接看到日志。

```bash
[error] [config] indentation level is too low
```

这里启动报了个错，翻译过来意思就是缩进太短了，编辑 config 发现是从 ZO WEB界面复制出来的 OUTPUT 跟原来的 INPUT 缩进不一样，给 OUTPUT 再多缩进一些跟 INPUT 对齐就行了。

### Gin

一个简单的可以实现我们的目的的 Demo

main.go

```go
package main

import (
    "fmt"
    "github.com/gin-gonic/gin"
    "io"
    "net"
    "time"
)

// 确保 FluentBit 实现了 io.Writer 接口
var _ io.Writer = (*FluentBit)(nil)

var (
    // DefaultInterval 默认与 fluent-bit 的连接断开后，重连的间隔时间
    DefaultInterval = time.Second
    // Address fluent-bit 的地址
    Address = "127.0.0.1:5170"
)

// FluentBit 表示与 fluent-bit 的连接
type FluentBit struct {
    address        string
    conn           net.Conn
    intervalTicker <-chan time.Time
}

var fb FluentBit

// 初始化时连接 fluent-bit，并且设置 gin 的日志输出到 fluent-bit
func init() {
    // 不需要控制台输出，这里禁用控制台输出的颜色
    gin.DisableConsoleColor()
    fb = FluentBit{
        address:        Address,
        intervalTicker: time.Tick(DefaultInterval),
    }
    fb.Connect()
    gin.DefaultWriter = &fb
}

func main() {
    // 检测与 fluent-bit 的连接是否断开，如果断开则重连
    go func() {
        for {
            <-fb.intervalTicker
            if _, err := fb.Write([]byte("")); err != nil {
                fb.Reconnect()
            }
        }
    }()

    r := gin.New()
    // 使用自定义的日志格式，这里用 json 格式，方便 fluent-bit 解析
    // ZO 会根据接收到日志的时间自动添加 _timestamp 字段
    // 这里我们自己指定一个 _timestamp，这样 ZO 会直接使用我们添加的
    r.Use(gin.LoggerWithFormatter(func(param gin.LogFormatterParams) string {
        return fmt.Sprintf(`{"_timestamp":"%d","log":"%s -  %s %s %s %d %s %s %s"}`,
            param.TimeStamp.UnixNano(),
            param.ClientIP,
            param.Method,
            param.Path,
            param.Request.Proto,
            param.StatusCode,
            param.Latency,
            param.Request.UserAgent(),
            param.ErrorMessage,
        )
    }))
    r.Use(gin.Recovery())
    // 简单的 ping pong 用来测试
    r.GET("/ping", func(c *gin.Context) {
        c.String(200, "pong")
    })

    _ = r.Run(":8080")
}

func (f *FluentBit) Write(b []byte) (n int, err error) {
    return f.conn.Write(b)
}
func (f *FluentBit) Close() {
    _ = f.conn.Close()
}

func (f *FluentBit) Connect() {
    conn, _ := net.Dial("tcp", f.address)
    f.conn = conn
}

func (f *FluentBit) Reconnect() {
    f.Close()
    f.Connect()
    _, _ = f.Write([]byte(`{"message":"reconnect to fluent-bit"}`))
}
```

这里为 `FluentBit` 实现的方法，它们的方法对象都是 `*FluentBit`，原因是在 GO 中都是值传递，如果方法对象是 `FluentBit` ，那么在调用方法的时候 `FluentBit` 结构体会被克隆一份然后在克隆出来的`FluentBit`上执行方法。

而考虑到可能 tcp 会断连（比如 fluent-bit 挂掉了），我们需要重连 tcp，如果方法对象是 `FluentBit` 则会导致重连后 gin 拿到的 Writer 仍旧是断连前的那个（因为重连这个操作是在克隆出来的结构体上执行的，新的连接是放入的克隆出来的结构体内）。

这里小记一下：**在 go 中，当一个方法接收者是具有一定的共享属性时，要使用指针接收者，或者方法接收者不是由普通的  go 内建类型构成的，比较庞大，这时克隆一份接收者的成本过高，也应该使用指针接收者**

### 测试是否成功

```bash
go run .
```

控制台没有日志输出，因为日志直接写入 fluent-bit 了。

再看 fluent-bit

```textile
[ warn] [input:tcp:tcp.0] invalid JSON message, skipping
```

新增了这样的日志输出，这是因为 gin 启动的时候会打印一些东西，那些不是 json。无关紧要。

接下来测试一下:

```bash
$ curl -i localhost:8080/ping
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Date: Tue, 25 Apr 2023 12:02:37 GMT
Content-Length: 4

pong
```

这时 fluent-bit 中新打印了一条:

```textile
[ info] [output:http:http.0] localhost:5080, HTTP status=200
{"code":200,"status":[{"name":"default","successful":1,"failed":0}]}
```

说明成功了。

去 ZO WEB 页面看看:

![gin_zo1](https://raw.githubusercontent.com/m4n5ter/m4n5ter.github.io/main/assets/gin_zo1.png)

可以看到成功了，到这里我们的目的就达成了。

## 如何替换 Logstash

上面我们是直接将日志发送到 fluent-bit，再由 fluent-bit 直接发送给 ZO，但是如果有大量不同应用程序的日志需要搜集，那么日志会非常混乱不好管理，这时就需要在发送给 ZO 之前清洗日志了，而 Logstash 就能干这个（fluent-bit 本身也有一定这方面能力），但是 Logstash 也是蛮重量级的，我们不想要消耗这么多资源怎么办？

go-zero 的作者 kevwan 有开源一个项目，叫做 [go-stash](https://github.com/kevwan/go-stash) ，能够从 kafka 摄取数据并加工后发送到 elasticsearch ，并且吞吐量能达到 logstash 的 5倍。

但是但是吧，Kafka 也是蛮重量级的，如果业务环境中本身没有 kafka ，为了这个特地多加一个 kafka，系统复杂度上去了不说，还有额外运维成本，还有服务器资源成本。那有没有轻量级的替代呢？也有，CNCF的云原生消息系统 [NATS](https://nats.io/) 可以替换 kafka，并且 NATS 也能直接和 fluent-bit 联动。

## 写一个自己的 "go-stash"

我们还是需要一个类似于 go-stash 的东西，来加工 fluent-bit 收集的日志，需要能够直接支持从 fluent-bit 接收数据，和从 NATS 接收数据并加工处理后发送到 ZO。

需求:

1. input 至少支持 fluent-bit/NATS

2. output 至少支持 ZO/ES（ZO 兼容 ES API）

3. 需要够轻量，使用 GO 或 RUST 开发

接下来我会抽时间持续更新这块，若大体完成我会在 github 上开源这个工具。
