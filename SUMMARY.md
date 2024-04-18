# Table of contents

* [Introduction](introduction.md)

## 一. 理论 - 论文解析
* [In Search of an Understandable Consensus Algorithm (Extended Version)](chapter1/paper1.md)
* [CONSENSUS: BRIDGING THEORY AND PRACTICE](chapter1/paper2.md)

## 二. 实现 - braft 源码解析
* [简介](chapter2/introduction.md)
* [初始化](chapter2/init)
    * [章节概览](chapter2/init_overview.md)
    * [节点初始化](chapter2/init_node.md)
    * [Multi-Raft](chapter2/init_multi_raft.md)
* [选举](chapter2/election)
    * [章节概览](chapter2/election_overview.md)
    * [选主流程](chapter2/election.md)
    * [选主优化](chapter2/election_optimization.md)
* [日志复制](chapter2/log_replication)
    * [章节概览](chapter2/log_overview.md)
    * [复制流程](chapter2/log_replication.md)
    * [日志管理](chapter2/log_manager.md)
    * [日志存储](chapter2/log_storage.md)
* [快照](chapter2/snapshot)
    * [章节概览](chapter2/snapshot_overview.md)
    * [创建快照](chapter2/snapshot_save.md)
    * [安装快照](chapter2/snapshot_install.md)
    * [加载快照](chapter2/snapshot_load.md)
* [控制节点](chapter2/cli)
    * [章节概览](chapter2/cli_overview.md)
    * [节点配置变更](chapter2/cli_configuration_change.md)
    * [重置节点列表](chapter2/cli_reset_peer.md)
    * [转移 Leader](chapter2/cli_change_leader.md)
* [使用指南](chapter2/guide)
    * [章节概览](chapter2/guide_overview.md)
    * [使用指南](chapter2/guide_use.md)
    * [性能分析](chapter2/guide_perf.md)
    * [可观测性](chapter2/guide_trace.md)
    * [超时时间](chapter2/guide_timeout.md)
* [总结](chapter2/summary.md)

## 三. 应用 - braft 在 CurveFS 中的应用