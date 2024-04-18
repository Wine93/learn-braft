流程详解
===

流程概览
---

1. 用户调用 `transfer_leadership_to` 接口转移 Leader
2. 若当前 Leader 正在转移或有配置变更正在进行中，则返回 `EBUSY`
3. Leader 将自身状态设为 `Transferring`，此时所有的 `apply` 会报错，Leader 停止写入，无新增日志
4. 判断目标节点日志是否和当前 Leader 一样多：
   * 4.1 如果是的话，向目标节点发送 `TimeoutNow` 请求开始重新选举
   * 4.2 否则，继续在后台向 Follower 同步日志，每成功同步一批日志，就重复步骤 4
5. 调用用户状态机的 `on_leader_stop`
6. 启动转移超时定时器；该步骤后，`transfer_leadership_to` 接口返回成功，但变更仍在继续
7. 节点收到 `TimeoutNow` 请求后，并行进行以下 2 件事：
   * 7.1 设置 `TimeoutNow` 响应中的 `term` 为自身 `term` 加一，并发送响应
   * 7.2 立马变为 `Candidate` 并自增 `term` 进行选举（跳过 `PreVote` 阶段）
8. Leader 收到 `TimeoutNow` 响应后，发现目标节点的 `term` 比自身大，则开始 `step_down` 成 Follower
9. 如果在 `election_timeout_ms` 时间内 Leader 没有 `step_down`，则取消迁移操作：
    * 调用状态机的 `on_leader_start` 继续领导当前任期，此时 Leader 的 `term` 并未改变
    * 将自身状态变为 `Leader`，并开始重新接受写入请求

相关 RPC
---

```proto
message TimeoutNowRequest {
    required string group_id = 1;
    required string server_id = 2;
    required string peer_id = 3;
    required int64 term = 4;
    optional bool old_leader_stepped_down = 5;
}

message TimeoutNowResponse {
    required int64 term = 1;
    required bool success = 2;
}

service RaftService {
    rpc timeout_now(TimeoutNowRequest) returns (TimeoutNowResponse);
};
```

相关接口
---

```cpp
class Node {
public:
    // Try transferring leadership to |peer|.
    // If peer is ANY_PEER, a proper follower will be chosen as the leader for
    // the next term.
    // Returns 0 on success, -1 otherwise.
    int transfer_leadership_to(const PeerId& peer);
};
```

阶段一：开始转移 Leader
===

transfer_leadership_to
---

用户调用 `transfer_leadership_to` 开始转移 Leader，其流程见以下注释：

```cpp
int NodeImpl::transfer_leadership_to(const PeerId& peer) {
    std::unique_lock<raft_mutex_t> lck(_mutex);
    // (1) 如果当前 Leader 正在转移，则返回 EBUSY
    if (_state != STATE_LEADER) {
        ...
        return _state == STATE_TRANSFERRING ? EBUSY : EPERM;
    }

    // (2) 如果当前 Leader 有配置变更正在进行中，则返回 EBUSY
    if (_conf_ctx.is_busy() /*FIXME: make this expression more readable*/) {
        // It's very messy to deal with the case when the |peer| received
        // TimeoutNowRequest and increase the term while somehow another leader
        // which was not replicated with the newest configuration has been
        // elected. If no add_peer with this very |peer| is to be invoked ever
        // after nor this peer is to be killed, this peer will spin in the voting
        // procedure and make the each new leader stepped down when the peer
        // reached vote timedout and it starts to vote (because it will increase
        // the term of the group)
        // To make things simple, refuse the operation and force users to
        // invoke transfer_leadership_to after configuration changing is
        // completed so that the peer's configuration is up-to-date when it
        // receives the TimeOutNowRequest.
        ...
        return EBUSY;
    }

    // (3) 对目标 PeerId 做一些检查
    PeerId peer_id = peer;
    // if peer_id is ANY_PEER(0.0.0.0:0:0), the peer with the largest
    // last_log_id will be selected.
    if (peer_id == ANY_PEER) {
        ...
        // find the next candidate which is the most possible to become new leader
        if (_replicator_group.find_the_next_candidate(&peer_id, _conf) != 0) {
            return -1;
        }
    }
    if (peer_id == _server_id) {
        ...
        return 0;
    }

    if (!_conf.contains(peer_id)) {
        ...
        return EINVAL;
    }

    // (4) 记录当前日志的的 lastLogIndex，
    //     并调用 ReplicatorGroup::transfer_leadership_to 来判断目标节点的日志是否和当前 Leader 一样多
    //     见以下 <判断差距>
    const int64_t last_log_index = _log_manager->last_log_index();
    const int rc = _replicator_group.transfer_leadership_to(peer_id, last_log_index);
    ...
    // (5) 将状态设为 Transferring，此时将停止写入，所有 apply 都会报错
    //     见以下 <停止写入>
    _state = STATE_TRANSFERRING;
    ...
    // (6) 调用用户状态机的 on_leader_stop
    _fsm_caller->on_leader_stop(status);
    ...
    // (7) 启动转移超时定时器，若在 election_timeout_ms 时间内 Leader 没有 step_down,
    //     则调用 on_transfer_timeout
    if (bthread_timer_add(&_transfer_timer,
                       butil::milliseconds_from_now(_options.election_timeout_ms),
                       on_transfer_timeout, _stop_transfer_arg) != 0) {
        ...
        return -1;
    }
    return 0;
}
```

判断差距
---

`ReplicatorGroup::transfer_leadership_to` 会在 `ReplicatorGroup` 找到目标节点的 `Replicator`，然后调用其 `_transfer_leadership` 函数。

在 `_transfer_leadership` 中主要判断目标节点的日志是否和当前 Leader 一样多：
* 如果是的话，直接进入阶段三发送 `TimeoutNow` 请求进行重新选举
* 否则，保存 `lastLogIndex` 为 `timeoutNowIndex`，并进入阶段二继续同步日志

```cpp
int ReplicatorGroup::transfer_leadership_to(
        const PeerId& peer, int64_t log_index) {
    // (1) 找到对应的 Replicator
    std::map<PeerId, ReplicatorIdAndStatus>::const_iterator iter = _rmap.find(peer);
    if (iter == _rmap.end()) {
        return EINVAL;
    }
    ...
    return Replicator::transfer_leadership(rid, log_index);
}

int Replicator::transfer_leadership(ReplicatorId id, int64_t log_index) {
    ...
    return r->_transfer_leadership(log_index);
}

// (2) 最终调用 Replicator 的 _transfer_leadership
int Replicator::_transfer_leadership(int64_t log_index) {
    /*
     * int64_t _min_flying_index() {  // 返回已经成功同步的 logIndex
     *     return _next_index - _flying_append_entries_size;
     * }
     *
     * (3) 如果目标节点日志和当前 Leader 一样多，
     *     则直接进入发送 TimeoutNow 请求进行重新选举，
     *     详见 <阶段三：重新选举>
     */
    if (_has_succeeded && _min_flying_index() > log_index) {
        // _id is unlock in _send_timeout_now
        _send_timeout_now(true, false);
        return 0;
    }

    // (3.1) 否则，保存 lastLogIndex 为 timeoutNowIndex
    //       继续同步日志，每通过一批日志，就重判差距
    //       详见 <阶段二：同步日志>
    // Register log_index so that _on_rpc_returned trigger
    // _send_timeout_now if _min_flying_index reaches log_index
    _timeout_now_index = log_index;
    return 0;
}
```

停止写入
---

```cpp
void NodeImpl::apply(LogEntryAndClosure tasks[], size_t size) {
    ...
    std::unique_lock<raft_mutex_t> lck(_mutex);
    ...
    // (1) 如果当前 Leader 正在转移,则将 status 设为 EPERM
    if (_state != STATE_LEADER || reject_new_user_logs) {
        if (_state == STATE_LEADER && reject_new_user_logs) {
            ...
        } else if (_state != STATE_TRANSFERRING) {
            st.set_error(EPERM, "is not leader");
        } else {
            ...
        }
        ...
        // (2) 回调所有 Task 的 Closure
        for (size_t i = 0; i < size; ++i) {
            tasks[i].entry->Release();
            if (tasks[i].done) {
                tasks[i].done->status() = st;
                run_closure_in_bthread(tasks[i].done);
            }
        }
        return;
    }
    ...
}
```

阶段二：同步日志
===

`Replicator` 的作用就是不断调用 `_send_entries` 同步日志，直至 Follower 同步了 Leader 的全部日志才在后台等待，详见[<4.1 复制流程>](/ch04/4.1/replicate.md)：

同步日志
---

```cpp
void Replicator::_send_entries() {
    ...
    // (1) 更新 nextIndex 以及 flyingAppendEntriesSize
    _next_index += request->entries_size();
    _flying_append_entries_size += request->entries_size();
    ...
    // (2) 向 Follower 发送 AppendEntries 请求来同步日志，
    //     并设置响应回调函数为 _on_rpc_returned
    google::protobuf::Closure* done = brpc::NewCallback(
                _on_rpc_returned, _id.value, cntl.get(),
                request.get(), response.get(), butil::monotonic_time_ms());
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
    ...
}
```

重判差距
---

每当收到日志复制的 `AppendEntries` 响应，都会回调 `_on_rpc_returned`：

```cpp
void Replicator::_on_rpc_returned(ReplicatorId id, brpc::Controller* cntl,
                     AppendEntriesRequest* request,
                     AppendEntriesResponse* response,
                     int64_t rpc_send_time) {
    ...
    r->_has_succeeded = true;
    ...
    // int64_t _min_flying_index() {  // 返回已经成功同步的 logIndex
    //     return _next_index - _flying_append_entries_size;
    // }
    // (1) 如果 Follower 已经同步了 Leader 的全部日志，则发送 TimeoutNow 请求进行重新选举
    if (r->_timeout_now_index > 0 && r->_timeout_now_index < r->_min_flying_index()) {
        r->_send_timeout_now(false, false);
    }
    // (2) 如果没有则继续发送日志
    r->_send_entries();
    return;
}
```

阶段三：重新选举
===

发送请求
---

Leader 调用 `_send_timeout_now` 向目标节点发送 `TimeoutNow` 请求：

```cpp
void Replicator::_send_timeout_now(bool unlock_id, bool old_leader_stepped_down,
                                   int timeout_ms) {
    TimeoutNowRequest* request = new TimeoutNowRequest;
    TimeoutNowResponse* response = new TimeoutNowResponse;
    ...
    request->set_peer_id(_options.peer_id.to_string());
    ...
    RaftService_Stub stub(&_sending_channel);
    ::google::protobuf::Closure* done = brpc::NewCallback(
            _on_timeout_now_returned, _id.value, cntl, request, response,
            old_leader_stepped_down);
    stub.timeout_now(cntl, request, response, done);
    ...
}
```

处理请求
---

Follower 收到 `TimeoutNow` 请求后，会调用 `handle_timeout_now_request` 处理请求，具体流程见以下注释：

```cpp
void NodeImpl::handle_timeout_now_request(brpc::Controller* controller,
                                          const TimeoutNowRequest* request,
                                          TimeoutNowResponse* response,
                                          google::protobuf::Closure* done) {
    brpc::ClosureGuard done_guard(done);
    ...
    // (1) 设置响应中的 term 为自身 term 加一
    // Increase term to make leader step down
    response->set_term(_current_term + 1);
    ...
    response->set_success(true);
    // (2) 并行发送响应和调用 elect_self 进行选举
    // Parallelize Response and election
    run_closure_in_bthread(done_guard.release());
    elect_self(&lck, request->old_leader_stepped_down());
    ...
}
```

收到响应
---

Leader 在收到 `TimeoutNow` 响应后，会调用 `_on_timeout_now_returned` 进行 `step_down`，并在 `step_down` 函数中移除转移超时定时器：

```cpp
void Replicator::_on_timeout_now_returned(
                ReplicatorId id, brpc::Controller* cntl,
                TimeoutNowRequest* request,
                TimeoutNowResponse* response,
                bool old_leader_stepped_down) {
    ...
    // (1) 如果目标节点的 term 比自身大，则开始 step_down 成 Follower
    if (response->term() > r->_options.term) {
        NodeImpl *node_impl = r->_options.node;
        ...
        node_impl->increase_term_to(response->term(), status);
        ...
        return;
    }
    ...
}

int NodeImpl::increase_term_to(int64_t new_term, const butil::Status& status) {
    ...
    // (2) 调用 step_down
    step_down(new_term, false, status);
    return 0;
}

void NodeImpl::step_down(const int64_t term, bool wakeup_a_candidate,
                         const butil::Status& status) {
    // (3) 变为 Follower
    // soft state in memory
    _state = STATE_FOLLOWER;

    // (4) 投票给目标节点，并持久化 term 和 votedFor
    // meta state
    if (term > _current_term) {
        //TODO: outof lock
        butil::Status status = _meta_storage->
                    set_term_and_votedfor(term, _voted_id, _v_group_id);
        ...
    }
    ...
    // (5) 删除转移超时定时器
    const int rc = bthread_timer_del(_transfer_timer);
}
```

其他：转移超时
===

如果在 `election_timeout_ms` 时间内 Leader 没有 `step_down`，则转移超时定时器就会超时，调用 `on_transfer_timeout` 进行处理：

```cpp
void on_transfer_timeout(void* arg) {
    ...
    // (1) 调用 handle_transfer_timeout
    a->node->handle_transfer_timeout(a->term, a->peer);
    ...
}

void NodeImpl::handle_transfer_timeout(int64_t term, const PeerId& peer) {
    ...
    if (term == _current_term) {
        // (2) 取消迁移操作
        _replicator_group.stop_transfer_leadership(peer);
        if (_state == STATE_TRANSFERRING) {
            // (3) 调用用户状态机的 on_leader_start 继续领导当前任期
            //     Leader 的 term 并未改变
            _fsm_caller->on_leader_start(term, _leader_lease.lease_epoch());
            // (4) 将自身角色设为 Leader
            _state = STATE_LEADER;
            ...
        }
    }
}

int Replicator::stop_transfer_leadership(ReplicatorId id) {
    ...
    // (2.1) 将 timeoutNowIndex 设为 0，
    //       这样每次同步日志后就不需要判断差距，也不会再发送 TimeoutNow 请求
    r->_timeout_now_index = 0;
    ...
    return 0;
}
```