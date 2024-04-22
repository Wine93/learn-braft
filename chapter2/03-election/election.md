
TODO

同一个节点在一个 term 内只能投票给一个人？ 会不会出现瓜分选票的情况？

整体流程
---
看代码画一张流程图，leader 和 follower 分别（上下）

![alt text](未命名文件-2.png)

具体实现
---
这里整理下

具体实现
---
![预先](election.svg)

### 阶段一: 预投票（*Pre-Vote*）

![alt text](未命名文件.png)

![alt text](未命名文件-1.png)


### 阶段二:

![alt text](vote.png)



### 阶段三: 等待投票结果


分辨
===

* https://zhuanlan.zhihu.com/p/366661148

```
原集群commit了更多的日志，或Term增加。A重新加入集群后，由于没有足够新的日志或Term较小，它的选举请求将会被绝大多数节点拒绝。A将会在Leader节点控制下，重新成为Follower。
// 会成为 follower 吗？term 比它大？

原集群没有commit更多的日志且Term没有增加。A重新加入集群后，将可能引发集群中新一轮的选举，并产生一个新的Leader。
```

## Leader Lease

```cpp
int64_t NodeImpl::last_leader_active_timestamp(const Configuration& conf) {

}
```

> 关于 replicator
> 1. 作为客户端
> 2.


TODO
---
* lease 的作用? 如何利用其实现 follower 读？ // follower lease 吧?
* leader lease 与 follower lease 的区别？
// https://cn.pingcap.com/blog/lease-read/
* raft_enable_leader_lease
* https://github.com/baidu/braft/issues/154#issuecomment-517727551


# prevote，怎么触发选举，leader 这么降为 follower?
* leader 向心跳
* 广播？

# vote timeoout 作用

`tradeoff`

failure 的时间和

https://infocenter.sybase.com/help/index.jsp?topic=/com.sybase.infocenter.dc31654.1570/html/sag1/sag1148.htm



```cpp
int NodeImpl::init(const NodeOptions& options)
    CHECK_EQ(0, _election_timer.init(this, options.election_timeout_ms * 2));
    CHECK_EQ(0, _vote_timer.init(this, options.election_timeout_ms * 2 + options.max_clock_drift_ms));
    CHECK_EQ(0, _stepdown_timer.init(this, options.election_timeout_ms));
    CHECK_EQ(0, _snapshot_timer.init(this, options.snapshot_interval_s * 1000));
```

```cpp
void ElectionTimer::run()
    _node->handle_election_timeout();
```

```cpp
void NodeImpl::handle_election_timeout()
    return pre_vote(&lck, triggered);
```

```cpp
void NodeImpl::pre_vote(std::unique_lock<raft_mutex_t>* lck, bool triggered)
    for (std::set<PeerId>::const_iterator iter = peers.begin();
         iter != peers.end(); ++iter) {

        RaftService_Stub stub(&channel);
        stub.pre_vote(&done->cntl, &done->request, &done->response, done);
    }

    grant_self(&_pre_vote_ctx, lck);
```

```cpp
void NodeImpl::grant_self(VoteBallotCtx* vote_ctx, std::unique_lock<raft_mutex_t>* lck)
        if (vote_ctx == &_pre_vote_ctx) {
            elect_self(lck);
        } else {
            become_leader();
        }
```


```cpp
void NodeImpl::elect_self(std::unique_lock<raft_mutex_t>* lck, bool old_leader_stepped_down)

    request_peers_to_vote(peers, _vote_ctx.disrupted_leader());

```

```cpp
void NodeImpl::request_peers_to_vote(const std::set<PeerId>& peers,
                                     const DisruptedLeader& disrupted_leader)
    for (std::set<PeerId>::const_iterator
        iter = peers.begin(); iter != peers.end(); ++iter) {

        OnRequestVoteRPCDone* done =
            new OnRequestVoteRPCDone(*iter, _current_term, _vote_ctx.version(), this);

        RaftService_Stub stub(&channel);
        stub.request_vote(&done->cntl, &done->request, &done->response, done);
    }
```

```cpp
struct OnRequestVoteRPCDone : public google::protobuf::Closure {
    void Run() {
        node->handle_request_vote_response(peer, term, ctx_version, response);
    }
}
```

```cpp
void NodeImpl::handle_request_vote_response(const PeerId& peer_id, const int64_t term,
                                            const int64_t ctx_version,
                                            const RequestVoteResponse& response)
    if (response.granted()) {
        if (_vote_ctx.granted()) {
            return become_leader();
        }
    }
```


```cpp
void NodeImpl::become_leader()

    for (std::set<PeerId>::const_iterator iter = peers.begin();
         iter != peers.end(); ++iter) {

        _replicator_group.add_replicator(*iter);
    }
```

```
int ReplicatorGroup::add_replicator(const PeerId& peer)
    Replicator::start(options, &rid);

    _rmap[peer] = { rid, options.replicator_status };
```

