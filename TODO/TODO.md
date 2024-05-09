TODO
---

* 介绍下 3 大队列，是不是顺序的保证

```cpp
bthread::ExecutionQueueId<StableClosure*> _disk_queue;  // LogManager
bthread::ExecutionQueueId<ApplyTask> _queue_id; // FSMCaller
bthread::ExecutionQueueId<LogEntryAndClosure> _apply_queue_id; // node
```

* apply 下去的 done 啥时候会设置成成功会失败？

* 重启的时候啥时候回放日志？ 看下 on_snapshot_load

* commitIndex / applid_index 是否会持久化 // 不会持久化

* 3 个节点，任务提交后如果只有一个节点了怎么办？会返回给业务层

```cpp
RAFT需要三种不同的持久存储, 分别是:

RaftMetaStorage, 用来存放一些RAFT算法自身的状态数据， 比如term, vote_for等信息.
LogStorage, 用来存放用户提交的WAL
SnapshotStorage, 用来存放用户的Snapshot以及元信息.
```


快照
===

* 简介
* 整体设计
* 整体实现
* 实现细节

### 必看：https://www.zhihu.com/people/yang-zhi-hu-65-47/posts

*

* 快照异步是不是有问题？会不会导致日志丢失？比如说快照没打成功，就把对应的日志给删除了???
* 各个 follower 各自打快照有啥问题吗？ // 选择什么时机打？如果打的时候 leader 向 follower 发送快照呢？
* raft meta 的作用
* 定时器什么时候启动的
* 快照的加载与保存流程？
* 接受 leader 的快照，本地的快照这么办？删除吗？ // 现在看是指保存一个，以 snapshot_00000000012345 命名? 为什么需要这个后缀
* 快照是否保存两份

* 什么时候会触发安装快照？
    * 一个是 leader 发现 follower 落后太多
    * 重启的时候
* follower 在接受快照的时候是不是得停止其它任务
* 是否
* raft_do_snapshot_min_index_gap // 这玩意官方文档好像没有？可以提个 PR 补充下


* `do_snapshot_save` 和其他 task 任务不同的是?
    * clouser 的调用交给用户了，不会等待 done->run 完成，允许你做异步操作

## 快照元数据信息
* 文件列表，以及每个文件对应的校验值


快照执行器
===

* 如果阻塞，可能或导致快照一致阻塞

```

在成为 *Leader* 后，主要做这几件事：

* 通过发送空的 *AppendEntries* 请求来确定各个 *Follower* 的 *next_index*；
* 啥时候可以服务?
* next_index 和 committed_index 是这么确认的？


no-op
---

no-op 的作用

* https://github.com/baidu/braft/issues?q=is%3Aissue+noop
* https://zhuanlan.zhihu.com/p/362679439
* https://zhuanlan.zhihu.com/p/30706032

幽灵复现

* https://mp.weixin.qq.com/s?__biz=MzIzOTU0NTQ0MA==&mid=2247494453&idx=1&sn=17b8a97fe9490d94e14b6a0583222837&scene=21#wechat_redirect
* https://zhuanlan.zhihu.com/p/652849109

braft log recovery
* https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md#log-recovery


心跳
---


Check Quorum
===

简介
---

最后那个按照etcd的实现就是check quorum，可以再clear一点，解决的问题 1可以解决疑似脑裂问题，就是raft 以为自己是主，但是因为失联，其实主早就不是他了，这个时候check quorum 只是一种优化，因为正常情况下有新leader出现导致写不进去client会重试 2.其实最关键的是解决失联中非对称网络隔离，例如leader 可以一直发ae 给follower ，但是leader收不到ack ，这种极其严重，会导致raft 组一直不可用，所以必须自己检测自己

当一个节点成为 *Leader* 时，会启动一个 `StepdownTimer`，该定时器会定期检查集群中的节点是否存活，如果发现集群中的节点不足以组成 *Quorum*，则会主动降级。

实现
---

```cpp
void NodeImpl::become_leader() {
    ...
    _stepdown_timer.start();
}
```


Follower Lease
===


leader lease
===

* 作用：解决 leader 还是 leader 的问题，防止 stale read
* API 介绍，使用

