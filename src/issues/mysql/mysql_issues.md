## [ERROR] Can't start server: can't check PID filepath: No such file or directory

> “强制关机后 `mysql` （指物理机上部署，非容器）怎么启动不了了？！”

这个报错是由于强制关机导致 `mysql` `pid` 文件丢失，查看 `mysql` 配置文件，找到 `pid` 文件位置，创建 `pid` 文件所在的目录并 `chown mysql:mysql <目录>`即可

## [ERROR] Fatal error: Please read "Security" section of the manual to find out how to run mysqld as root

```bash
# 指定用户来启动 mysqld
mysqld --user=<USER> 
```



## 备份脚本例子

```zsh
#!/bin/bash
mysql_user="xxxx"
mysql_password="xxxx"
mysql_host="xxxx"
mysql_port="xxxx"
backup_dir=/opt/mysql_backup

dt=`date +'%Y%m%d_%H%M'`
echo "Backup Begin Date:" $(date +"%Y-%m-%d %H:%M:%S")

# 备份全部数据库
mysqldump -h$mysql_host -P$mysql_port -u$mysql_user -p$mysql_password -R -E --all-databases --single-transaction > $backup_dir/mysql_backup_$dt.sql

find $backup_dir -mtime +7 -type f -name '*.sql' -exec rm -rf {} \;
echo "Backup Succeed Date:" $(date +"%Y-%m-%d %H:%M:%S")

```


