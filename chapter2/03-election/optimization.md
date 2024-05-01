选主优化
===

概览
===

*braft* 在实现选举的时候做了一些优化，这些优化点归纳起来主要为了实现以下几个目的：
* **快**: 减少选举时间，让集群尽快产生 *Leader*（e.g. 选举时间随机化、*Wakeup Candidate*）
* **稳定**：当集群中有 *Leader* 时，尽可能保持稳定，减少没必要的选主（e.g. 心跳、Pre-Vote）
* **脑裂**：当出现脑裂时，尽可能减少双主存在的时间，并让客户端有办法感知，（e.g. Check Quorum、Leader Lease）

对于基于 *braft* 实现的系统来说，拥有这些优化可以提升系统整体的可用性。

| 优化项         | 说明                    |
|:---------------|:------------------------|
| 选举超时随机化 | 避免选举风暴            |
| 检查 qurom     | 避免脑裂                |
| leader lease   | 避免 leader 还是 leader |


优化点 1：选举超时时间随机化
===

介绍
---

如果多个节点同时 *election timeout* 而触发选举，可能会造成选票被瓜分，导致无法选出 *Leader*，从而触发下一轮选举。为此，可以将 *election timeout* 随机化，减少选票被瓜分的可能性。但是在极端情况下，多个节点可能
                                       

![极端情况：随机的 election timeout](image.png)

实现
---

优化点 2： Wakeup Candidate
===

当一个 Leader 正常退出时，它会选择一个节点向其发送 `TimeoutNow` 请求并让该节点立马进行选举，无需等待超时。

```cpp
```

pre vote
---

check quorum
---

```cpp
void NodeImpl::become_leader() {
}
```

```cpp
void StepdownTimer::run() {
    _node->handle_stepdown_timeout();
}

void NodeImpl::handle_stepdown_timeout() {
    BAIDU_SCOPED_LOCK(_mutex);

    // check state
    if (_state > STATE_TRANSFERRING) {
        BRAFT_VLOG << "node " << _group_id << ":" << _server_id
            << " term " << _current_term << " stop stepdown_timer"
            << " state is " << state2str(_state);
        return;
    }
    check_witness(_conf.conf);
    int64_t now = butil::monotonic_time_ms();
    check_dead_nodes(_conf.conf, now);
    if (!_conf.old_conf.empty()) {
        check_dead_nodes(_conf.old_conf, now);
    }
}
```

```cpp
void NodeImpl::check_dead_nodes(const Configuration& conf, int64_t now_ms) {
    std::vector<PeerId> peers;
    conf.list_peers(&peers);
    size_t alive_count = 0;
    Configuration dead_nodes;  // for easily print
    for (size_t i = 0; i < peers.size(); i++) {
        if (peers[i] == _server_id) {
            ++alive_count;
            continue;
        }

        if (now_ms - _replicator_group.last_rpc_send_timestamp(peers[i])
                <= _options.election_timeout_ms) {
            ++alive_count;
            continue;
        }
        dead_nodes.add_peer(peers[i]);
    }
    if (alive_count >= peers.size() / 2 + 1) {
        return;
    }
    LOG(WARNING) << "node " << node_id()
                 << " term " << _current_term
                 << " steps down when alive nodes don't satisfy quorum"
                    " dead_nodes: " << dead_nodes
                 << " conf: " << conf;
    butil::Status status;
    status.set_error(ERAFTTIMEDOUT, "Majority of the group dies");
    step_down(_current_term, false, status);
}

```

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

参考
===

* [分布式一致性 Raft 与 JRaft](https://www.sofastack.tech/projects/sofa-jraft/consistency-raft-jraft/)
* [关于 DDIA 上对 Raft 协议的这种极端场景的描述，要如何理解？](https://www.zhihu.com/question/483967518)
* [Symmetric network partitioning](https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md#symmetric-network-partitioning)
* [Raft在网络分区时leader选举的一个疑问？](https://www.zhihu.com/question/302761390)
* [Raft 必备的优化手段（一）：Leader Election 篇](https://zhuanlan.zhihu.com/p/639480562)
* [Raft 笔记(四) – Leader election](https://youjiali1995.github.io/raft/etcd-raft-leader-election/)
* [raft: implement leader steps down #3866](https://github.com/etcd-io/etcd/issues/3866)

* [共识协议优质资料汇总（paxos，raft）](https://zhuanlan.zhihu.com/p/628681520)
