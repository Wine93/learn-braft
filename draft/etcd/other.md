# 源码阅读 (0): 总纲

---

> **问题**

* [ ] 搞懂 paxos、raft、2pc 等分布式一致性算法区别、应用场景...
* [x] 搞清楚日志 log、计时等概念及在 raft 实现中的应用
* [x] leader 中的 log 并没有完全复制到任意一个 follower，此时 leader 就挂掉了，那么此后重新选举出来的 Leader 怎么拥护完整的 log 呢？（因为有一部分 log 没复制成功，leader 挂掉了）
    * leader 每条日志至少有一半节点接收了才 commit (才算提交成功，返回客户端), 而且每次选举肯定会保证接收日志最多的那个节点成为 leader，所以可以保证新选的 leader 拥有完整的日志
* [ ] raft 已经被 commit 的日志会不会被覆盖？

---

> **阅读顺序**

* [ ] raft 启动多个节点，进行选主，产生 leader 的过程
    * [ ] node 的添加
    * [ ] 跟踪下 raftLog 的变化的，它是如何持久化的
    * [x] 投票信息怎么通过网络传递的
    * [ ] 选举成功后怎么发送心跳包
    * [ ] 跟踪完整的 Ready 与 Advance 过程，跟踪 commit 及 apply 2 个游标的更新
        * [ ] 如果是投票类 (MsgVote) 信息是否会 commit?
    * [ ] hardstate 和 softstate 作用及区别
    * [ ] 快照到底是个啥玩意
* [ ] kvstore 接受客户端请求，进行操作的流程（中间涉及到了 raft 算法的配合）
    * [ ] 日志的 append
* [ ] 其他的问题
    * [ ] msg 和 entries 的区别? 分别怎么添加进去的? (msg: r.send()?)
    * [ ] 一个 raft 节点需要处理哪几类信息？比如来自本地的 MsgProp 和来自其他节点的 MsgVote 等, 做清晰的区别？

---

> **模块分类**

##### 日志持久化:

* **storage.go**:  持久化日志保存模块，以 interface 的方式定义了实现的方式,并基于内存实现了 memoryStorage 用于存储日志数据
* **log.go**:  raft 算法日志模块的逻辑
* **log_unstable.go**:  raft 算法的日志缓存，日志优先写缓存，待状态稳定后进行持久化

##### 节点:
* **node.go**:  raft 集群节点行为的实现，定义了各节点通信方式
* **process.go**:  从 leader 的角度，为每个 follower 维护一个子状态机，根据状态的切换决定 leader 该发什么消息给 follower

##### raft 算法:
* **raft.go**:  raft 算法的具体逻辑实现，每个节点都有一个 raft 实例
* **read_only.go**:  实现了线性一致读 (linearizable read)，线性一致读要求读请求读到最新提交的数据。针对 raft 存在的 stale read (多 leader 场景)，此模块通过 ReadIndex 的方式保证了一致性

---

# 源码阅读 (00): 总纲

目录
====

* [相关概念](#相关概念)
    * [时钟](#时钟)
    * [gossip 协议](gossip-协议)
* [TODO](#todo)

相关概念
=======

时钟
====

* [Netflix/dynomite](https://github.com/Netflix/dynomite)
* [Dynamo 涉及的算法和协议](https://www.letiantian.me/2014-06-16-dynamo-algorithm-protocol/)
* [Dynamo 论文介绍](https://catkang.github.io/2016/05/27/dynamo.html)
* [分布式系统：Lamport 逻辑时钟](https://blog.xiaohansong.com/lamport-logic-clock.html)
* [分布式系统：向量时钟](https://blog.xiaohansong.com/lamport-logic-clock.html)

gossip 协议
===========

* [分布式原理: 一文了解 gossip 协议](https://www.iteblog.com/archives/2505.html)
* [raft 算法和 gossip 协议](https://www.backendcloud.cn/2017/11/12/raft-gossip/)
* [hashicorp/memberlist](https://github.com/hashicorp/memberlist)
* 看下 redis cluster 中 gaossip 算法的应用
* gossip 协议如何计算总和

TODO
====

* [ ] 了解 multi-raft 的实现
* [ ] 阅读知乎所有的分布式专栏, 选取几个精彩的专栏和文章
    * [分布式系统思考和相关论文](https://zhuanlan.zhihu.com/little-ds)
    * [分布式，存储](https://zhuanlan.zhihu.com/dsystem)
        * [etcd raft如何实现成员变更](https://zhuanlan.zhihu.com/p/27908888)
    * [分布式点滴](https://zhuanlan.zhihu.com/learn-distributed-system)
    * [分布式与存储](https://zhuanlan.zhihu.com/codeit)
        * leveldb 源码解析
    * [分布式系统理论](https://zhuanlan.zhihu.com/distributed)

* 搞清楚以下的概念
    * [ ] 时钟？raft 是否存在? 还是只是在 pasxos 中存在
    * [ ] 什么是 2pc? raft 中是否存在?
        * [ ] 分布式事务的解决方案? (2PC 算一个)
    * [ ] 什么是 ZAB 算法?
    * [ ] 大致了解 WAL 实现?
        * [ ] https://zhuanlan.zhihu.com/p/24900322
        * [ ] 看下 etcd 中 wal 的应用

* 整理下 raft 相关的问题?
    * 参考 https://www.cnblogs.com/cchust/p/5634782.html

---

> **问题**

* [ ] 搞懂 paxos、raft、2pc 等分布式一致性算法区别、应用场景...
* [x] 搞清楚日志 log、计时等概念及在 raft 实现中的应用
* [x] leader 中的 log 并没有完全复制到任意一个 follower，此时 leader 就挂掉了，那么此后重新选举出来的 Leader 怎么拥护完整的 log 呢？（因为有一部分 log 没复制成功，leader 挂掉了）
    * leader 每条日志至少有一半节点接收了才 commit (才算提交成功，返回客户端), 而且每次选举肯定会保证接收日志最多的那个节点成为 leader，所以可以保证新选的 leader 拥有完整的日志
* [ ] raft 已经被 commit 的日志会不会被覆盖？
* [ ] 关于日志安全性的问题？
    * 什么时候日志会被覆盖或者会被裁剪?

---

> **阅读顺序**

* [ ] raft 启动多个节点，进行选主，产生 leader 的过程
    * [ ] node 的添加
    * [ ] 跟踪下 raftLog 的变化的，它是如何持久化的
    * [x] 投票信息怎么通过网络传递的
    * [ ] 选举成功后怎么发送心跳包
    * [ ] 跟踪完整的 Ready 与 Advance 过程，跟踪 commit 及 apply 2 个游标的更新
        * [ ] 如果是投票类 (MsgVote) 信息是否会 commit?
    * [ ] hardstate 和 softstate 作用及区别
    * [ ] 快照到底是个啥玩意
* [ ] kvstore 接受客户端请求，进行操作的流程（中间涉及到了 raft 算法的配合）
    * [ ] 日志的 append
* [ ] 其他的问题
    * [ ] msg 和 entries 的区别? 分别怎么添加进去的? (msg: r.send()?)
    * [ ] 一个 raft 节点需要处理哪几类信息？比如来自本地的 MsgProp 和来自其他节点的 MsgVote 等, 做清晰的区别？

---

> **模块分类**

##### 日志持久化:

* **storage.go**:  持久化日志保存模块，以 interface 的方式定义了实现的方式,并基于内存实现了 memoryStorage 用于存储日志数据
* **log.go**:  raft 算法日志模块的逻辑
* **log_unstable.go**:  raft 算法的日志缓存，日志优先写缓存，待状态稳定后进行持久化

##### 节点:
* **node.go**:  raft 集群节点行为的实现，定义了各节点通信方式
* **process.go**:  从 leader 的角度，为每个 follower 维护一个子状态机，根据状态的切换决定 leader 该发什么消息给 follower

##### raft 算法:
* **raft.go**:  raft 算法的具体逻辑实现，每个节点都有一个 raft 实例
* **read_only.go**:  实现了线性一致读 (linearizable read)，线性一致读要求读请求读到最新提交的数据。针对 raft 存在的 stale read (多 leader 场景)，此模块通过 ReadIndex 的方式保证了一致性

---
