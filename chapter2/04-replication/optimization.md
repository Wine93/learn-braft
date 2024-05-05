

Batch
===

Pipeline
===

![串行与 Pipeline](image/pipeline-2.svg)

Append Log Parallelly
===

![](image/append_parallel.svg)

针对慢节点

Asynchronous Apply
===

Others
===

raft sync
---

braft 对于每次日志落盘都会进行 `sync`，如果业务对于数据丢失的容忍度比较高，可以选择将配置项 `raft_sync` 设置为 `Flase`，这将有助于。另外值得一提的是，影响日志可靠性

控制日志，

https://github.com/baidu/braft/issues?q=is%3Aissue+sync

参考
===

* [braft 性能优化](https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md#%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
* [braft 复制模型](https://github.com/baidu/braft/blob/master/docs/cn/replication.md)
* [TiKV 功能介绍 – Raft 的优化](https://cn.pingcap.com/blog/optimizing-raft-in-tikv/)
* [Raft 必备的优化手段（二）：Log Replication & others](https://zhuanlan.zhihu.com/p/668511529)
