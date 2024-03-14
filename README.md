learn-raft
===

Raft 理论、实现与应用

第一部分（理论）- *论文解析*
---

### 论文总结
* [In Search of an Understandable Consensus Algorithm (Extended Version)](chapter1/paper1.md)
* [CONSENSUS: BRIDGING THEORY AND PRACTICE](chapter1/paper1.md)

第二部分（实现）- *braft 源码解析*
---

### 基础概念


### 选举
* [服务初始化](#)
* [选举流程](#)
* [投票箱](#)
* [心跳](#)

### 日志复制
* [整体概览]()
* [复制模型]()
* [日志管理]()
* [日志存储]()

### 快照
* [本地快照]()
* [本地快照]()
* [本地快照]()

### 配置变更
* [整体流程]()

### 其他
* [multi-raft 的实现]()

### cheetsheat
*

第三部分（应用）- *braft 在 CurveFS 中的应用*
---

* [心跳与调度](blog/curvefs/)
* [快照](blog/curvefs/)

说明
---
由于本人水平有限，文中难免有错误，还请各位读者批评指正。
