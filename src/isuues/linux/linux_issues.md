
## Out-Of-Memory(OOM)

OOM 是一种比较容易遇见（尤其是类似 java 程序这种对内存资源的需求比较大的）的问题。常见的解决方式有：
- 对程序进行相关配置或调整（如配置 jvm 参数或分析并优化程序代码）
- 调整内核参数

### 调整内核参数

这里我主要介绍第二种方式。在 linux 中会出现 `Out of Memory: Killed process 12345 (postgres).` 这种情况的原因就是 linux 内核支持 memory overcommit，overcommit 就是内核允许进程过量使用内存，比如在只有 1G 内存可用的环境中进程申请了 1.1G。

而 linux overcommit 的默认内核策略是允许进程少量 overcommit，拒绝大量 overcommit，即试探性的允许 overcommit。这个内核参数是 `vm.overcommit_memory`，默认值是 `0`。`1` 表示 "Always overcommit"，总是允许 overcommit。最后一个值是 `2`，也是一般我们需要的模式，"Don't overcommit" ，这个模式将会直接拒绝 overcommit，它可以有效减少发生 OOM 的概率，但绝不是不会发生。

```bash
# 查看 vm.overcommit_memory 的当前值
$ sysctl vm.overcommit_memory
# 更改为 2
$ sysctl -w vm.overcommit_memory=2
# 在 /etc/sysctl.conf 中放置等效条目，来持久化该配置
$ echo "vm.overcommit_memory=2" >> /etc/sysctl.conf
```
如果配置了 `vm.overcommit_memory=2`，那么通常希望也配置 `vm.overcommit_ratio` ，这个参数的单位是百分比。配置它的原因是，“Don't overcommit” 这个模式不允许 commit 超过 `swap` + **一个可配置的基于物理内存的百分比的量**（默认值是 50），而这个可配置的量就是 `vm.overcommit_ratio`，至于配置方法同上文的 bash 内容。

### 较危险的方法

当然，还有一种方法可以直接让内核的 `OOM killer` 不会把我们的进程当做目标，那就是配置 `/proc/<PID>/oom_score_adj`，`<PID>` 表示相应进程的 ID。`echo -1000 > /proc/<PID>/oom_score_adj`将我们的进程的 oom_score_adj 设置为 -1000，表示让 `OOM killer` 不要把这个进程当做目标。原理是每个进程都会根据实施情况被打上一个`oom_score` 分数，当系统的内存不够用时 `OOM killer` 会寻找 `/proc/<PID>/oom_score` 高的进程并将其杀死。 而打分时 `oom_score_adj` 的值会直接影响 `oom_score` 的值，`oom_score_adj` 的范围是 `-1000 ~ 1000` ，当 `oom_score_adj` 的值越低，`oom_score` 被打的分也就会越低。设置 `oom_score_ajd` 为 `-1000` 时 `OOM killer` 就永远不会光顾这个进程了。

这个方法危险的原因则是，当 `OOM killer` 永远不会去杀死一个进程，而这个进程又需要大量内存资源时，当内存不够用，`OOM killer` 会去杀死其它分高的进程，这是十分危险的。 
