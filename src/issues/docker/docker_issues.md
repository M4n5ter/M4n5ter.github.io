## error creating overlay mount to...

>  "强制关机后再次重启机器后发现一些 docker 容器出现了 `error creating overlay mount to......` 并且无法启动"

该问题是与 `selinux` 相关的，有一 issue 与该问题相关[#2430](https://github.com/coreos/bugs/issues/2340)

查看 `/etc/selinux/config` ，`selinux` 状态为 `disabled` ，在将其设置为 `permissive` 后重启机器解决了该问题。

[#2430](https://github.com/coreos/bugs/issues/2340) 中有老哥提到禁用 `selinux` 解决了他的问题，但是我的情况是本身 `selinux` 状态就是 `disabled` ，在修改为 `permissive` 后解决了该问题。



## Error response from daemon: Cannot restart container drone: driver failed programming external connectivity on endpoint drone (7df4fc8df955812c87a501d92249f4e9eb41ee820908569ca3cb98544b2bad9c):  (iptables failed: iptables --wait -t nat -A DOCKER -p tcp -d 0/0 --dport 3001 -j DNAT --to-destination 172.17.0.7:80 ! -i docker0: iptables: No chain/target/match by that name.

 这种错误一般是由于 dockerd 定义的 iptables 自定义链因为一些原因（最常见的就是发生在操作防火墙后）被清除了。此时把 docker 重启一下，让它重新生产自定义链即可。
