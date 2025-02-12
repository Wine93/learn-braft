learn-raft
===

braft 源码解析（完整版）<sup>[在线阅读](https://wine93.gitbook.io/learn-raft)</sup>

目录
---

* [前言](introduction.md)
* [1. 简介](ch01/introduction.md)
* [2. 启动节点](ch02/README.md)
  * [2.1 启动流程](ch02/2.1/init.md)
  * [2.2 Multi-Raft](ch02/2.2/multi_raft.md)
* [3. Leader 选举](ch03/README.md)
  * [3.1 选主流程](ch03/3.1/election.md)
  * [3.2 选主优化](ch03/3.2/optimization.md)
  * [3.3 Witness 与 Learner](ch03/3.3/witness_learner.md)
* [4. 日志复制](ch04/README.md)
  * [4.1 复制流程](ch04/4.1/replicate.md)
  * [4.2 复制优化](ch04/4.2/optimization.md)
  * [4.3 日志存储](ch04/4.3/log_storage.md)
* [5 快照](ch05/README.md)
  * [5.1 创建快照](ch05/5.1/save_snapshot.md)
  * [5.2 安装快照](ch05/5.2/install_snapshot.md)
  * [5.3 加载快照](ch05/5.3/load_snapshot.md)
* [6. 控制节点](ch06/README.md)
  * [6.1 节点配置变更](ch06/6.1/configuration_change.md)
  * [6.2 重置节点列表](ch06/6.2/reset_peer.md)
  * [6.3 转移 Leader](ch06/6.3/change_leader.md)
* [附录](appendix.md)

说明
---
由于本人水平有限，文中可能出现一些纰漏或错误的地方，欢迎大家以提交 [issue][issue] 或 [PR][pull-request] 的方式进行更正和完善。如果文中有些描述不清晰，或者你有任何疑问和建议，都可以在 [issue][issue] 中告诉我。

[issue]: https://github.com/Wine93/learn-raft/issues
[pull-request]: https://github.com/Wine93/learn-raft/pulls
