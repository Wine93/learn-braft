选主优化
===


优化点
---
分别阐述下各优化点，其他作用，简述下其简要实现，这里可以补充一些图

具体实现
---

主要是贴代码



表格：

| 选主算法 | 作用 | 适用场景 | |
| :--- | :--- | :--- | :--- |
| 选举超时随机化 | 避免选举风暴 | 选举时间不确定 | 高并发场景 |
| 检查 qurom | 避免脑裂 | 选举时间不确定 | 高并发场景 |
| leader lease | 避免 leader 还是 leader | 选举时间不确定 | 高并发场景 |

选举超时随机化
---

pre vote
---

check quorum
---

```cpp
void NodeImpl::become_leader() {
    ...
    _state = STATE_LEADER;
    ...
    _stepdown_timer.start();
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

check qurom
---


leader lease
---

* 作用：解决 leader 还是 leader 的问题，防止 stale read
* API 介绍，使用

参考
===

* [Raft 必备的优化手段（一）：Leader Election 篇](https://zhuanlan.zhihu.com/p/639480562)

* [共识协议优质资料汇总（paxos，raft）](https://zhuanlan.zhihu.com/p/628681520)
