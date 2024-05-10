章节概览
===

* [3.1 选主流程](chapter2/election.md): 介绍了 Leader 选举流程和流程中一些关键点，以及相关源码解析
* [3.2 选主优化](chapter2/election_optimization.md): 介绍了 braft 选主
* [3.3 Learnner 与 Witness](chapter2/learner_witness.md): Raft 除了 `Follower`、`Candidate`、`Leader` 这 3 个角色外，还引入了 `Learner` 与 `Witness`，该小节主要介绍其作用，以及在 braft 中的具体实现
