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