## nmap-netcat(nc)

nc 是由 C 编写的非常强悍的网络工具，`nc` 主要有 4 个版本，gnu nc （2004 年停止维护，也叫 nc traditional）， openbsd nc（重写了 gnu nc，在正常维护），nmap-netcat（也叫 ncat，由 nmap 重写 nc traditional，被称为 21 世纪的 netcat，也是功能最多的 nc）。

下面我介绍的就是 nmap-netcat。

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
