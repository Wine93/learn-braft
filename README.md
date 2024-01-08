learn-raft
===

raft 理论、实现与应用

目录
===

raft 论文解析
---

* [选主](blog/paper.md)
* [配置变更](blog/paper.md)

braft 源码解析
---

* [整体实现](blog/overview.md)
* [服务初始化](blog/init.md)
* [选举](blog/election.md)
* [心跳](blog/heartbeat.md)
* [日志复制](blog/log_replication.md)
* [快照](blog/snapshot.md)
* [配置变更](blog/configuration_change.md)
* [multi-raft 实现](blog/multi_raft.md)

// https://zhuanlan.zhihu.com/p/169904153
// https://zhuanlan.zhihu.com/p/169840204

// 博士论文翻译：https://github.com/OneSizeFitsQuorum/raft-thesis-zh_cn/tree/master

curvefs 中 raft 的
