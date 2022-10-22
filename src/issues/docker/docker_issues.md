## error creating overlay mount to...

>  "强制关机后再次重启机器后发现一些 docker 容器出现了 `error creating overlay mount to......` 并且无法启动"

该问题是与 `selinux` 相关的，有一 issue 与该问题相关[#2430](https://github.com/coreos/bugs/issues/2340)

查看 `/etc/selinux/config` ，`selinux` 状态为 `disabled` ，在将其设置为 `permissive` 后重启机器解决了该问题。

[#2430](https://github.com/coreos/bugs/issues/2340) 中有老哥提到禁用 `selinux` 解决了他的问题，但是我的情况是本身 `selinux` 状态就是 `disabled` ，在修改为 `permissive` 后解决了该问题。


