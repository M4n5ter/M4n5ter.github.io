# 网络知识随心记

## tcp 粘包

tcp 会出现粘包是由于 tcp 本身并不是基于消息的协议，tcp 是基于流的，所以在 tcp 协议的视角里，一切都是流，所做的优化都是将数据视作流为前提的。

当使用 tcp 协议发送多个小的数据包时，tcp 会在数据包到达接收方时进行优化，将多个小的包合并为一个包后存放在接收方缓冲区，这就是粘包的由来。粘包会导致应用层解析数据比较困难，因为多个数据包被合并为了一个数据包。针对这个问题可以有很多解决方案：

- **固定长度**：每个数据包都采用固定的长度，接收方可以根据这个固定长度分割接收到的数据流。

- **定界符或分隔符**：发送方在每个数据包的末尾添加一个特殊的字符或字符串，接收方根据这些分隔符来识别数据包的边界。

- **长度字段**：在每个数据包的开始部分添加一个字段来指示数据包的长度，接收方先读取长度字段以确定数据包的大小。

- **自定义协议**：定义包含起始标识、数据长度、数据内容和校验等字段的复杂数据包结构，便于接收方进行解析和校验。



## 由队头阻塞引出的一系列思考

### 传输层队头阻塞

tcp 有队头阻塞问题，这是传输层的队头阻塞，但是应用层同样页会发生队头阻塞。

tcp 的队头阻塞问题来自于 tcp 的设计，tcp 为了保证传输的可靠性会确保传输过程中字节流的顺序和完整性，并遵循 tcp 的丢包重传等机制。如果一个包丢失了，TCP层面就会触发重传机制，并按照原有的顺序等待丢失的包被正确接收后才能继续处理之后的数据。因此会产生队头因各种各样的原因没能被正确接收时，即使后续的数据已经到达，依旧需要等待队头被正确接收。

> 思考：在传输层会出现队头阻塞，那么应用层也会有吗？



### 应用层队头阻塞

最常见的就是 http/1.1 协议，在 http/1.1 中，在一个连接上只有当完整的组装出了协议报文后，才能处理下一个报文。所以当一个 http/1.1 报文是残缺的时候，需要等待重传，直到被报文组装完成后，该连接才能继续处理后续的报文。

> 思考：传输层的 TCP/UDP 根深蒂固，并且我们无法干预传输层协议的处理。那么 http/1.1 有队头阻塞问题，这个队头阻塞是出现在应用层，应用层的协议相比传输层的协议更容易处理，应用层的队头阻塞有什么解决办法吗？



### HTTP/2 针对应用层队头阻塞的处理

http/2 协议不再是一个基于文本的协议，http header 会被压缩来提高传输效率，并且所有的信息都会被编码成二进制。而 http/2 解决 http/1.1 的队头阻塞问题采用的方式是将报文拆分成一个个的二进制帧，从而实现 tcp 之上的多路复用，http/2 的传输以二进制帧为最小单位，并且在一个 tcp 连接上可以交叉发送不同 http/2 报文的二进制帧，也就是说当一个帧出现问题时，并不影响其它的帧，当帧到达接收方时会进行组装。因此应用层的队头阻塞问题也就不再存在了，因为 http/2 不会等待某个报文被组装完成后才能继续处理下一个报文，http/2 会尽可能的将得到的二进制帧进行组装，拼成一个个完整的报文。

但是！这也仅仅是解决了应用层的队头阻塞问题，http/2 依旧是一个基于 tcp 的协议，传输层的 tcp 队头阻塞问题是无法解决的。而 http/3 或者说 quic 的出现解决了传输层的队头阻塞问题，因为直接构建于更简单的 udp 协议之上（越简单即意味着限制越小，更容易在其之上进行自定义设计）。

> 思考：HTTP/2 可以全面替代 HTTP/1.1吗？HTTP 永远都会被队头阻塞问题困扰吗？



### HTTP/2 可以全面替代 HTTP/1.1吗

首先 HTTP2 一定不是银弹。比如：

HTTP/2 的多路复用在带宽较低的网络环境中可能会使得单个请求之间出现竞争，导致一些重要资源的加载被延迟。在这种场景下，1.1 的队头阻塞可能倒不如是一种优点，因为它可以保证至少一个请求是在持续完成的。



HTTP/2 是基于长连接的：
在 HTTP/1.1 中，虽然支持了 keep-alive 机制来复用 TCP 连接发起多个请求，但它仍然有一定的限制，如一次只能处理一个请求/响应，后一个请求必须等前一个完成才能开始。这就是著名的“队头阻塞”问题。

而 HTTP/2 被设计来克服这些限制，并更有效地利用 TCP 连接。它引入了多路复用的概念，能够让多个请求/响应在同一个 TCP 连接上并行交错发送，每个请求/响应都是独立的流，并共享这一条连接。长连接加上多路复用，极大地提高了性能和效率，降低了延迟，改善了网络带宽的利用。

在HTTP/2 中，通常当客户端向服务器发起第一个请求时，它们之间建立一个 TCP 连接，然后对于后续的所有HTTP请求，只要条件允许，会复用这个已建立的TCP连接，而不是每次请求都重新建立一个。这个长连接会保持打开状态直到客户端或服务器决定关闭它，或者由于某种原因（如超时或错误）而断开。

但是！！长连接也是存在问题的，计算机世界没有银弹！

1. **资源占用**：长连接会占用服务器资源，每个连接都会消耗一定的内存和处理能力。如果服务器同时维护大量的长连接，可能会导致资源紧张，降低服务器的处理能力。

2. **不活跃连接**：一些长连接可能在大部分时间内都是空闲的，不活跃的连接占据资源却没有实际的数据传输，造成了资源的浪费。

3. **超时管理**：需要合理设置连接的超时时间，过短可能导致频繁的连接建立和断开，增加开销；过长则可能使那些暂时不活跃的连接占用资源过久。

4. **扩展性问题**：随着客户端数量的增加，长连接可能导致服务器的并发连接数迅速增长，对服务器的扩展性造成挑战。

5. **连接复用的逻辑复杂性**：正确管理复用的逻辑相对复杂，需要确保连接的正确关闭和异常处理，以及多个请求之间不会共享应该是私有的数据。

6. **网络状态变化**：长连接在网络条件变化时可能会变得不稳定，比如在移动网络中，用户的设备在切换网络或信号不稳定时，长连接可能会断开需要重新连接。



HTTP/2 总体上是对 HTTP/1.1 的更好的改进，但是全面替代在短时间内是不可能的，再说后面还有 HTTP3 等着呢，想要全面普及，最需要的就是时间。



### QUIC 是如何解决队头阻塞问题的

首先 quic 是基于 udp 的，而 udp 本身是无连接的协议，既然无连接，那么在传输层就根本没有队头阻塞这个问题。

> 思考：但是 udp 不是"不可靠"吗？



### QUIC 是如何保证可靠传输的

虽然因为基于 udp 协议不需要考虑队头阻塞这种问题，但是又因为 udp 本身不提供可靠性，也就是 quic 在传输层是没有可靠性保证的，所以自然保证可靠性的任务从传输层转移到了应用层，quic 协议本身需要保证传输的可靠性。因此 quic 引入了一套机制：

- **独立的流**：quic 中的数据传输通过独立的“流”进行，每个流相当于是一个独立的通道。即使某个流中的一个或多个包丢失，也只会影响到该具体流的传输，不会阻塞其他流中的数据。这就允许其他流的数据包继续被接收和处理，从而避免了传输层的队头阻塞。

- **快速重新传输**：quic 实现了更快速的丢包检测和重新传输机制。当数据丢失时，quic 可以快速识别该情况并重新传输丢失的数据，这比TCP的相应机制（如“快速重传”）通常能更快地解决问题，从而降低了一个丢失的数据包对整个连接的影响。

- **没有TCP的3次握手**：quic 不使用TCP的三次握手机制来建立连接，而是使用更简洁的机制，在最理想的情况下只需要一个往返时间（RTT）就可以完成连接建立和安全协商的过程。

- **连接迁移和连接ID**：quic 允许连接迁移，即使底层的IP地址发生变化，它依然可以维持现有的应用层连接不中断。quic 为每个连接指派了一个唯一的连接ID（connection ID），这使得连接在网络环境发生变化时更加健壮。

QUIC通过这些设计减少了数据传输中的延迟和潜在的阻塞问题，特别适用于移动网络和变化网络环境。这使得QUIC非常吸引那些需要低延迟和高性能网络通信的应用，如实时通讯和在线游戏。

> 思考：QUIC 是如何做到初次连接只需要 1 RTT 的？HTTP2 需要 2 RTT，如果使用 HTTPS 还需要加上 TLS1.3 的 1 RTT，也就是 3 RTT
>
> 至于 TLS1.3 为什么只需要 1 RTT 可以见左侧 Other categories 栏目中的 REALITY，可以在开头就得到答案。

### QUIC 的 RTT 为什么这么少

quic 最低可以做到 0 RTT，即之前已经建立过连接的客户端可以 0 RTT 重新建立连接，而初次连接只需要 1 RTT 即可建立，并且这个 1 RTT 中还包含了 TLS1.3。

QUIC 协议本身直接整合了 TLS1.3，在建立连接的同时顺带完成了 TLS 握手。



初次连接的过程是这样的：

1. 客户端向服务端发送 Initial 包，包含加密握手信息的 Initial 包。这个包为此后的安全通信提供了必要的参数和密钥。
2. 服务端向客户端发送 Initial 包，同时不需要等待 RTT，直接继续发送加密的 Handshake 信息，这些包包含了进一步的密钥协商信息，以及 TLS 握手信息
3. 客户端顺利收到服务端发来的所有包后，所需要的信息便全部得到，连接至此成功建立。

当然这是顺利的情况下，只需要 1 RTT，如果中间出现了差错就不是了。

初次握手完成后，客户端和服务器各自保存该次会话的相关参数和生成的票据。这个票据用于实现 0-RTT。



0-RTT 发生的过程：
1. 使用 0-RTT 重连：
   - 客户端在与服务器重新建立连接时，可以使用从上次会话中得到的票据来启动 0-RTT 数据传输。
   - 客户端发送包含 0-RTT 数据的包，这些数据使用之前会话中协商的密钥进行加密。
   - 客户端同时发送继续 TLS 握手所需的新的握手信息以建立新的会话密钥（不需要等待 RTT）。
   - 这允许客户端在第一个数据包就传输加密的应用数据而不需要等待服务器响应，避免了一次完整的 RTT。

2. 服务器处理 0-RTT 数据：
   - 服务器收到客户端的 0-RTT 数据后，首先使用之前会话中协商出的参数解密数据。
   - 如果服务器接受 0-RTT 数据，它可以马上开始处理这些数据并做出响应。
   - 同时，服务器继续处理客户端发送的新的 TLS 握手信息，以便为当前会话建立新的安全参数。

限制和安全考虑：
   - 0-RTT 数据虽然便利，但也增加了重放攻击的风险；因此它通常仅应用于幂等性（即重复执行不会产生不同结果）的操作。
   - 服务器可能设置策略限制 0-RTT 数据的使用场景或根据风险评估拒绝处理 0-RTT 数据。
   - 客户端需准备好在服务器拒绝接受 0-RTT 数据时回滚到常规的握手过程。



> 思考：那么 QUIC 那么好，基于 QUIC 的 HTTP/3 的未来如何？



### HTTP/3 的未来

主要问题就是 UDP 流量被 ISP "区别对待"，尤其是中国大陆的 ISP。

相对于 TCP，UDP 的确曾经有过不那么“友好”的对待，这主要是因为以下几个原因：

1. 网络优化和管理：某些互联网服务提供商（ISP）可能针对常见的 TCP 流量如网页加载和文件下载进行了优化，而没有为 UDP 流量提供同样的优化，因为后者传统上更多用于视频流、VoIP通话等需要较少网络管理的应用。

2. 流量整形（QoS）：网络运营商和管理员可能对流量进行整形，限制UDP流量以优先保证TCP流量的质量。因为很多关键的互联网服务都建立在 TCP 之上，且历史上UDP流量更可能被关联到视频、游戏或 P2P 应用，这些应用可能不被认为是网络上的优先级服务。

3. 防火墙和 NAT 设备：很多网络环境中的防火墙和 NAT 设备可能默认阻止或限制 UDP 流量，以避免潜在的安全风险或滥用。因为 UDP 相比于 TCP 来说，更容易被用于 DDoS 攻击。

4. 可靠性和拥塞控制：TCP 自身内置了拥塞控制和数据重传等机制，而 UDP 则没有。一些网络运营商为了网络稳定性可能会更偏向于利用这些机制的 TCP 流量。

随着互联网技术的发展，尤其是由 IETF 推动 QUIC 协议的标准化，很多这些问题正在得到解决。例如，QUIC 内置了类似 TCP 的可靠性和拥塞控制机制，并且由于 QUIC 提供了更低延迟的连接建立和更好的性能，网络提供商和设备制造商也逐渐开始提供对 UDP 流量更好的支持。当今，很多ISP和企业网络已经适应了新的协议，改善了对 UDP 流量的处理。



## TCP 的拥塞控制

### TCP 慢启动(Slow Start)



TCP 慢启动是一种流量控制算法，用于在建立一个新的TCP连接时探测网络的拥塞程度。其工作方式如下：

当开始一个新的TCP连接时，慢启动初始化一个拥塞窗口（Congestion Window，CWND），这是还没有被网络确认可以通过的最大数据量。开始时，CWND 的值非常小，通常是 1 个最大报文段大小（Maximum Segment Size，MSS）。每当一个段被确认，CWND 增加一个 MSS 的大小，这样的增长是指数级的（因为对于每个确认的包，CWND 都会增加），直到发生丢包或者达到一个阈值（ssthresh，慢启动阈值）。

慢启动的目的是避免新连接立即发送大量数据包，在网络还没有准备好承受此流量的情况下可能导致网络拥堵。



### 拥塞避免、快速重传、快速恢复

一旦超过慢启动阈值，或者检测到丢包事件（例如超时），则 TCP 连接进入拥塞控制阶段。TCP拥塞控制有四个主要的算法组成：慢启动、拥塞避免（Congestion Avoidance）、快速重传（Fast Retransmit）和快速恢复（Fast Recovery）。

- **拥塞避免**：当拥塞窗口大小超过慢启动阈值，TCP 使用拥塞避免算法，它不再指数级增长CWND，而是逐步（线性地）增加，通常每个往返时间（RTT）增加一个MSS的大小。

- **快速重传**：当发送端接收到三个重复的 ACK 时（表示一个数据段丢失），它会进行快速重传，而不必等待一个超时事件触发。

- **快速恢复**：在快速重传之后，TCP 进入快速恢复阶段。在这个阶段，它假定丢失是由于网络中的短暂的拥塞造成的。CWND 被减半，并且 ssthresh 被设置为这个新的CWND值。



## TCP 的流量控制

TCP 流量控制（TCP flow control）是通信中的一种机制，它确保发送方不会过快地发送数据以至于接收方来不及处理。它是 TCP 协议中用来避免发送方溢出接收方的缓冲区的机制。流量控制能够保证两端的数据处理速率是配合一致的，不会因为接收方处理慢而导致数据丢失。

流量控制的核心是「窗口」概念，每个 TCP 连接两端都会维护一个接收窗口（receive window），其大小由接收方通告给发送方，用来告知发送方自己当前能接受的数据量。这个窗口大小是动态变化的，基于接收方当前的缓冲区使用情况决定的。

详细步骤如下：

1. 建立连接时的窗口大小：在 TCP 握手过程中，接收方会在 TCP header 的窗口大小字段告知发送方其接收窗口的初始大小。

2. 发送方根据窗口大小发送数据：发送方在发送数据时要检查这个窗口大小，并确保未被确认的数据量不会超过这个窗口大小。

3. 接收方根据处理能力更新窗口：随着接收方逐渐处理数据，它会根据自己的空闲缓冲区大小动态调整窗口大小，并在发送确认信息（ACKs）给发送方时告知新的窗口大小。

4. 发送方根据新窗口调整发送流量：发送方在接收到新的窗口大小信息后，会相应调整后续数据的发送量，进而实现流量控制。

当接收方的接收窗口为零时，发送方必须停止发送数据，并等待接收方的窗口更新。若接收方长时间不更新窗口，发送方会发送一个探测报文，以促使接收方更新窗口。