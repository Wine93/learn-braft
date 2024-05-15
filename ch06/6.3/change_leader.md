流程详解
===

流程概览
---

1. 用户调用 `transfer_leadership_to` 接口转移 Leader
2. 若当前 Leader 正在转移或有配置变更正在进行中，则返回 `EBUSY`
3. Leader 停止写入，这时候所有的apply会报错.
4. 继续向所有的 Follower 同步日志，当发现目标节点的日志已经和主一样多之后，向对应节点发起一个`TimeoutNow` RPC
5. 回调用户状态机的 `on_leader_stop`
6. 节点收到 `TimeoutNow` 请求后，立马变为 `Candidate` 并增加 `Term` 进行选举
7. Leader 收到 `TimeoutNow` 响应后, 开始 `step_down`
8. 如果在 `election_timeout_ms` 时间内主没有 `step_down`，会取消主迁移操作，开始重新接受写入请求

流程注解
---

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

```cpp
int NodeImpl::transfer_leadership_to(const PeerId& peer) {
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (_state != STATE_LEADER) {
        LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " is in state " << state2str(_state);
        return _state == STATE_TRANSFERRING ? EBUSY : EPERM;
    }
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
        LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " refused to transfer leadership to peer " << peer
                     << " when the leader is changing the configuration";
        return EBUSY;
    }

    PeerId peer_id = peer;
    // if peer_id is ANY_PEER(0.0.0.0:0:0), the peer with the largest
    // last_log_id will be selected.
    if (peer_id == ANY_PEER) {
        LOG(INFO) << "node " << _group_id << ":" << _server_id
                  << " starts to transfer leadership to any peer.";
        // find the next candidate which is the most possible to become new leader
        if (_replicator_group.find_the_next_candidate(&peer_id, _conf) != 0) {
            return -1;
        }
    }
    if (peer_id == _server_id) {
        LOG(INFO) << "node " << _group_id << ":" << _server_id
                  << " transfering leadership to self";
        return 0;
    }
    if (!_conf.contains(peer_id)) {
        LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " refused to transfer leadership to peer " << peer_id
                     << " which doesn't belong to " << _conf.conf;
        return EINVAL;
    }
    const int64_t last_log_index = _log_manager->last_log_index();
    const int rc = _replicator_group.transfer_leadership_to(peer_id, last_log_index);
    if (rc != 0) {
        if (rc == EINVAL) {
            LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " fail to transfer leadership, no such peer=" << peer_id;
        } else if (rc == EHOSTUNREACH) {
            LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " fail to transfer leadership, peer=" << peer_id
                     << " whose consecutive_error_times not 0.";
        } else {
            LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " fail to transfer leadership, peer=" << peer_id
                     << " err: " << berror(rc);
        }
        return rc;
    }
    _state = STATE_TRANSFERRING;
    butil::Status status;
    status.set_error(ETRANSFERLEADERSHIP, "Raft leader is transferring "
            "leadership to %s", peer_id.to_string().c_str());
    _leader_lease.on_leader_stop();
    _fsm_caller->on_leader_stop(status);
    LOG(INFO) << "node " << _group_id << ":" << _server_id
              << " starts to transfer leadership to " << peer_id;
    _stop_transfer_arg = new StopTransferArg(this, _current_term, peer_id);
    if (bthread_timer_add(&_transfer_timer,
                       butil::milliseconds_from_now(_options.election_timeout_ms),
                       on_transfer_timeout, _stop_transfer_arg) != 0) {
        lck.unlock();
        LOG(ERROR) << "Fail to add timer";
        on_transfer_timeout(_stop_transfer_arg);
        return -1;
    }
    return 0;
}
```

```cpp
int ReplicatorGroup::transfer_leadership_to(
        const PeerId& peer, int64_t log_index) {
    std::map<PeerId, ReplicatorIdAndStatus>::const_iterator iter = _rmap.find(peer);
    if (iter == _rmap.end()) {
        return EINVAL;
    }
    ReplicatorId rid = iter->second.id;
    const int consecutive_error_times = Replicator::get_consecutive_error_times(rid);
    if (consecutive_error_times > 0) {
        return EHOSTUNREACH;
    }
    return Replicator::transfer_leadership(rid, log_index);
}

```


```cpp
int Replicator::transfer_leadership(ReplicatorId id, int64_t log_index) {
    Replicator* r = NULL;
    bthread_id_t dummy = { id };
    const int rc = bthread_id_lock(dummy, (void**)&r);
    if (rc != 0) {
        return rc;
    }
    // dummy is unlock in _transfer_leadership
    return r->_transfer_leadership(log_index);
}

int Replicator::_transfer_leadership(int64_t log_index) {
    if (_has_succeeded && _min_flying_index() > log_index) {
        // _id is unlock in _send_timeout_now
        _send_timeout_now(true, false);
        return 0;
    }
    // Register log_index so that _on_rpc_returned trigger
    // _send_timeout_now if _min_flying_index reaches log_index
    _timeout_now_index = log_index;
    CHECK_EQ(0, bthread_id_unlock(_id)) << "Fail to unlock " << _id;
    return 0;
}

void Replicator::_on_rpc_returned(ReplicatorId id, brpc::Controller* cntl,
                     AppendEntriesRequest* request,
                     AppendEntriesResponse* response,
                     int64_t rpc_send_time) {
    ...
    r->_has_succeeded = true;
    r->_notify_on_caught_up(0, false);
    // dummy_id is unlock in _send_entries
    if (r->_timeout_now_index > 0 && r->_timeout_now_index < r->_min_flying_index()) {
        r->_send_timeout_now(false, false);
    }
    r->_send_entries();
    return;
}
```

阶段二：同步日志
===

阶段三：重新选举
===

发送 `TimeoutNow` 请求
---

```cpp
void Replicator::_send_timeout_now(bool unlock_id, bool old_leader_stepped_down,
                                   int timeout_ms) {
    TimeoutNowRequest* request = new TimeoutNowRequest;
    TimeoutNowResponse* response = new TimeoutNowResponse;
    request->set_term(_options.term);
    request->set_group_id(_options.group_id);
    request->set_server_id(_options.server_id.to_string());
    request->set_peer_id(_options.peer_id.to_string());
    request->set_old_leader_stepped_down(old_leader_stepped_down);
    brpc::Controller* cntl = new brpc::Controller;
    if (!old_leader_stepped_down) {
        // This RPC is issued by transfer_leadership, save this call_id so that
        // the RPC can be cancelled by stop.
        _timeout_now_in_fly = cntl->call_id();
        _timeout_now_index = 0;
    }
    if (timeout_ms > 0) {
        cntl->set_timeout_ms(timeout_ms);
    }
    RaftService_Stub stub(&_sending_channel);
    ::google::protobuf::Closure* done = brpc::NewCallback(
            _on_timeout_now_returned, _id.value, cntl, request, response,
            old_leader_stepped_down);
    stub.timeout_now(cntl, request, response, done);
    if (unlock_id) {
        CHECK_EQ(0, bthread_id_unlock(_id));
    }
}
```

处理 `TimeoutNow` 请求
---

```cpp
void NodeImpl::handle_timeout_now_request(brpc::Controller* controller,
                                          const TimeoutNowRequest* request,
                                          TimeoutNowResponse* response,
                                          google::protobuf::Closure* done) {
    brpc::ClosureGuard done_guard(done);
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (request->term() != _current_term) {
        const int64_t saved_current_term = _current_term;
        if (request->term() > _current_term) {
            butil::Status status;
            status.set_error(EHIGHERTERMREQUEST, "Raft node receives higher term request.");
            step_down(request->term(), false, status);
        }
        response->set_term(_current_term);
        response->set_success(false);
        lck.unlock();
        LOG(INFO) << "node " << _group_id << ":" << _server_id
                  << " received handle_timeout_now_request while _current_term="
                  << saved_current_term << " didn't match request_term="
                  << request->term();
        return;
    }
    if (_state != STATE_FOLLOWER) {
        const State saved_state = _state;
        const int64_t saved_term = _current_term;
        response->set_term(_current_term);
        response->set_success(false);
        lck.unlock();
        LOG(INFO) << "node " << _group_id << ":" << _server_id
                  << " received handle_timeout_now_request while state is "
                  << state2str(saved_state) << " at term=" << saved_term;
        return;
    }
    const butil::EndPoint remote_side = controller->remote_side();
    const int64_t saved_term = _current_term;
    if (FLAGS_raft_enable_leader_lease) {
        // We will disrupt the leader, don't let the old leader
        // step down.
        response->set_term(_current_term);
    } else {
        // Increase term to make leader step down
        response->set_term(_current_term + 1);
    }
    response->set_success(true);
    // Parallelize Response and election
    run_closure_in_bthread(done_guard.release());
    elect_self(&lck, request->old_leader_stepped_down());
    // Don't touch any mutable field after this point, it's likely out of the
    // critical section
    if (lck.owns_lock()) {
        lck.unlock();
    }
    // Note: don't touch controller, request, response, done anymore since they
    // were dereferenced at this point
    LOG(INFO) << "node " << _group_id << ":" << _server_id
              << " received handle_timeout_now_request from "
              << remote_side << " at term=" << saved_term;

}
```


Leader 收到 `TimeoutNow` 响应
---

```cpp
void Replicator::_on_timeout_now_returned(
                ReplicatorId id, brpc::Controller* cntl,
                TimeoutNowRequest* request,
                TimeoutNowResponse* response,
                bool old_leader_stepped_down) {
    std::unique_ptr<brpc::Controller> cntl_guard(cntl);
    std::unique_ptr<TimeoutNowRequest>  req_guard(request);
    std::unique_ptr<TimeoutNowResponse> res_guard(response);
    Replicator *r = NULL;
    bthread_id_t dummy_id = { id };
    if (bthread_id_lock(dummy_id, (void**)&r) != 0) {
        return;
    }

    std::stringstream ss;
    ss << "node " << r->_options.group_id << ":" << r->_options.server_id
       << " received TimeoutNowResponse from "
       << r->_options.peer_id;

    if (cntl->Failed()) {
        ss << " fail : " << cntl->ErrorText();
        BRAFT_VLOG << ss.str();

        if (old_leader_stepped_down) {
            r->_notify_on_caught_up(ESTOP, true);
            r->_destroy();
        } else {
            CHECK_EQ(0, bthread_id_unlock(dummy_id));
        }
        return;
    }
    ss << (response->success() ? " success " : "fail:");
    BRAFT_VLOG << ss.str();

    if (response->term() > r->_options.term) {
        NodeImpl *node_impl = r->_options.node;
        // Acquire a reference of Node here in case that Node is detroyed
        // after _notify_on_caught_up.
        node_impl->AddRef();
        r->_notify_on_caught_up(EPERM, true);
        butil::Status status;
        status.set_error(EHIGHERTERMRESPONSE, "Leader receives higher term "
                "timeout_now_response from peer:%s", r->_options.peer_id.to_string().c_str());
        r->_destroy();
        node_impl->increase_term_to(response->term(), status);
        node_impl->Release();
        return;
    }
    if (old_leader_stepped_down) {
        r->_notify_on_caught_up(ESTOP, true);
        r->_destroy();
    } else {
        CHECK_EQ(0, bthread_id_unlock(dummy_id));
    }
}
```

```cpp
int NodeImpl::increase_term_to(int64_t new_term, const butil::Status& status) {
    BAIDU_SCOPED_LOCK(_mutex);
    if (new_term <= _current_term) {
        return EINVAL;
    }
    step_down(new_term, false, status);
    return 0;
}
```

其他：转移 Leader 超时
===

```cpp
void on_transfer_timeout(void* arg) {
    StopTransferArg* a = (StopTransferArg*)arg;
    a->node->handle_transfer_timeout(a->term, a->peer);
    delete a;
}
```

```cpp
void NodeImpl::handle_transfer_timeout(int64_t term, const PeerId& peer) {
    LOG(INFO) << "node " << node_id()  << " failed to transfer leadership to peer="
              << peer << " : reached timeout";
    BAIDU_SCOPED_LOCK(_mutex);
    if (term == _current_term) {
        _replicator_group.stop_transfer_leadership(peer);
        if (_state == STATE_TRANSFERRING) {
            _leader_lease.on_leader_start(term);
            _fsm_caller->on_leader_start(term, _leader_lease.lease_epoch());
            _state = STATE_LEADER;
            _stop_transfer_arg = NULL;
        }
    }
}
```