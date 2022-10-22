## [ERROR] Can't start server: can't check PID filepath: No such file or directory

> “强制关机后 `mysql` （指物理机上部署，非容器）怎么启动不了了？！”

这个报错是由于强制关机导致 `mysql` `pid` 文件丢失，查看 `mysql` 配置文件，找到 `pid` 文件位置，创建 `pid` 文件所在的目录并 `chown mysql:mysql <目录>`即可



## [ERROR] Fatal error: Please read "Security" section of the manual to find out how to run mysqld as root

```bash
# 指定用户来启动 mysqld
mysqld --user=<USER> 
```


