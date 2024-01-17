learn-raft
===

raft 理论、实现与应用

第一章：理论
---

* [raft 论文解析 - 选主](blog/paper.md)
* [raft 论文解析 - 配置变更](blog/paper.md)
* [raft 论文解析 - 日志压缩](blog/paper.md)

第二章：实现
---

* [braft 源码解析 - 整体实现](blog/overview.md)
* [braft 源码解析 - 服务初始化](blog/init.md)
* [braft 源码解析 - 选举](blog/election.md)
* [braft 源码解析 - 心跳](blog/heartbeat.md)
* [braft 源码解析 - 日志复制](blog/log_replication.md)
* [braft 源码解析 - 快照](blog/snapshot.md)
* [braft 源码解析 - 配置变更](blog/configuration_change.md)
* [braft 源码解析 - multi-raft 实现](blog/multi_raft.md)

第三章：应用
---

* [braft 在 CurveFS 中的应用  - 心跳与调度](blog/curvefs/)
