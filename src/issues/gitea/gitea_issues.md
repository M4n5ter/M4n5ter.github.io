## Unable to create internal queue for XXX Error: Unable to create queue level for XXX with cfg ...

按 [#18917 中的一条回复](https://github.com/go-gitea/gitea/issues/18917#issuecomment-1052342880) 中，发现没有 `LOCK` 文件，然后删除了 `data/queues/common` 后解决了该问题。

该 issue 中提到推荐采用 `redis` 来做 `queue`，配置方式在 [Config Cheat Sheet - Docs](https://docs.gitea.io/en-us/config-cheat-sheet/#indexer-indexer)
