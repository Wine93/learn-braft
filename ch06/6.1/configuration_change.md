流程详解
===

流程概览
---

流程注解
---

新节点的配置
---

新节点启动的时候需要为空节点，否则可能需要脑裂
参考 https://github.com/baidu/braft/issues/303

分布式事务
---

相关 RPC
---

```proto
message AddPeerRequest {
    required string group_id = 1;
    required string leader_id = 2;
    required string peer_id = 3;
}

message AddPeerResponse {
    repeated string old_peers = 1;
    repeated string new_peers = 2;
}

message RemovePeerRequest {
    required string group_id = 1;
    required string leader_id = 2;
    required string peer_id = 3;
}

message RemovePeerResponse {
    repeated string old_peers = 1;
    repeated string new_peers = 2;
}

message ChangePeersRequest {
    required string group_id = 1;
    required string leader_id = 2;
    repeated string new_peers = 3;
}

message ChangePeersResponse {
    repeated string old_peers = 1;
    repeated string new_peers = 2;
}

service CliService {
    rpc add_peer(AddPeerRequest) returns (AddPeerResponse);
    rpc remove_peer(RemovePeerRequest) returns (RemovePeerResponse);
    rpc change_peers(ChangePeersRequest) returns (ChangePeersResponse);
};
```

阶段一：CaughtUp
===

```cpp
void NodeImpl::add_peer(const PeerId& peer, Closure* done) {
    BAIDU_SCOPED_LOCK(_mutex);
    Configuration new_conf = _conf.conf;
    new_conf.add_peer(peer);
    return unsafe_register_conf_change(_conf.conf, new_conf, done);
}

void NodeImpl::remove_peer(const PeerId& peer, Closure* done) {
    BAIDU_SCOPED_LOCK(_mutex);
    Configuration new_conf = _conf.conf;
    new_conf.remove_peer(peer);
    return unsafe_register_conf_change(_conf.conf, new_conf, done);
}

void NodeImpl::change_peers(const Configuration& new_peers, Closure* done) {
    BAIDU_SCOPED_LOCK(_mutex);
    return unsafe_register_conf_change(_conf.conf, new_peers, done);
}
```

```cpp
void NodeImpl::unsafe_register_conf_change(const Configuration& old_conf,
                                           const Configuration& new_conf,
                                           Closure* done) {
    ...
    if (_conf_ctx.is_busy()) {  // 已经有配置变更在进行了
        ...
        return;
    }
    ...
    return _conf_ctx.start(old_conf, new_conf, done);
}
```

将新老配置做 diff 获得新加入的节点列表：

```cpp
void NodeImpl::ConfigurationCtx::start(const Configuration& old_conf,
                                       const Configuration& new_conf,
                                       Closure* done) {
    ...
    new_conf.diffs(old_conf, &adding, &removing);
    _nchanges = adding.size() + removing.size();
    ...
    if (adding.empty()) {
        ss << ", begin removing.";
        LOG(INFO) << ss.str();
        return next_stage();
    }

    adding.list_peers(&_adding_peers);
    for (std::set<PeerId>::const_iterator iter
            = _adding_peers.begin(); iter != _adding_peers.end(); ++iter) {

        if (_node->_replicator_group.add_replicator(*iter) != 0) {
            ...
        }

        OnCaughtUp* caught_up = new OnCaughtUp(
                _node, _node->_current_term, *iter, _version);
        timespec due_time = butil::milliseconds_from_now(
                _node->_options.get_catchup_timeout_ms());
        if (_node->_replicator_group.wait_caughtup(
            *iter, _node->_options.catchup_margin, &due_time, caught_up) != 0) {
            ...
        }
    }

```

```cpp
int ReplicatorGroup::wait_caughtup(const PeerId& peer,
                                   int64_t max_margin, const timespec* due_time,
                                   CatchupClosure* done) {
    ...
    Replicator::wait_for_caught_up(rid, max_margin, due_time, done);
    return;
}

void Replicator::wait_for_caught_up(ReplicatorId id,
                                    int64_t max_margin,
                                    const timespec* due_time,
                                    CatchupClosure* done) {
    r->_catchup_closure = done;
}
```

```cpp
class OnCaughtUp : public CatchupClosure {
public:
    ...
    virtual void Run() {
        _node->on_caughtup(_peer, _term, _version, status());
        delete this;
    };
    ...
};
```

每次 send_entries 后都会 notify

```cpp
void Replicator::_on_rpc_returned(ReplicatorId id, brpc::Controller* cntl,
                     AppendEntriesRequest* request,
                     AppendEntriesResponse* response,
                     int64_t rpc_send_time) {
    std::unique_ptr<brpc::Controller> cntl_guard(cntl);
    std::unique_ptr<AppendEntriesRequest>  req_guard(request);
    std::unique_ptr<AppendEntriesResponse> res_guard(response);
    Replicator *r = NULL;
    bthread_id_t dummy_id = { id };
    const long start_time_us = butil::gettimeofday_us();
    if (bthread_id_lock(dummy_id, (void**)&r) != 0) {
        return;
    }

    std::stringstream ss;
    ss << "node " << r->_options.group_id << ":" << r->_options.server_id
       << " received AppendEntriesResponse from "
       << r->_options.peer_id << " prev_log_index " << request->prev_log_index()
       << " prev_log_term " << request->prev_log_term() << " count " << request->entries_size();

    bool valid_rpc = false;
    int64_t rpc_first_index = request->prev_log_index() + 1;
    int64_t min_flying_index = r->_min_flying_index();  // _next_index - _flying_append_entries_size
    CHECK_GT(min_flying_index, 0);

    for (std::deque<FlyingAppendEntriesRpc>::iterator rpc_it = r->_append_entries_in_fly.begin();
        rpc_it != r->_append_entries_in_fly.end(); ++rpc_it) {
        if (rpc_it->log_index > rpc_first_index) {
            break;
        }
        if (rpc_it->call_id == cntl->call_id()) {
            valid_rpc = true;
        }
    }
    if (!valid_rpc) {
        ss << " ignore invalid rpc";
        BRAFT_VLOG << ss.str();
        CHECK_EQ(0, bthread_id_unlock(r->_id)) << "Fail to unlock " << r->_id;
        return;
    }

    if (cntl->Failed()) {
        ss << " fail, sleep.";
        BRAFT_VLOG << ss.str();

        // TODO: Should it be VLOG?
        LOG_IF(WARNING, (r->_consecutive_error_times++) % 10 == 0)
                        << "Group " << r->_options.group_id
                        << " fail to issue RPC to " << r->_options.peer_id
                        << " _consecutive_error_times=" << r->_consecutive_error_times
                        << ", " << cntl->ErrorText();
        // If the follower crashes, any RPC to the follower fails immediately,
        // so we need to block the follower for a while instead of looping until
        // it comes back or be removed
        // dummy_id is unlock in block
        r->_reset_next_index();
        return r->_block(start_time_us, cntl->ErrorCode());
    }
    r->_consecutive_error_times = 0;
    if (!response->success()) {
        if (response->term() > r->_options.term) {
            BRAFT_VLOG << " fail, greater term " << response->term()
                       << " expect term " << r->_options.term;
            r->_reset_next_index();

            NodeImpl *node_impl = r->_options.node;
            // Acquire a reference of Node here in case that Node is destroyed
            // after _notify_on_caught_up.
            node_impl->AddRef();
            r->_notify_on_caught_up(EPERM, true);
            butil::Status status;
            status.set_error(EHIGHERTERMRESPONSE, "Leader receives higher term "
                    "%s from peer:%s", response->GetTypeName().c_str(), r->_options.peer_id.to_string().c_str());
            r->_destroy();
            node_impl->increase_term_to(response->term(), status);
            node_impl->Release();
            return;
        }
        ss << " fail, find next_index remote last_log_index " << response->last_log_index()
           << " local next_index " << r->_next_index
           << " rpc prev_log_index " << request->prev_log_index();
        BRAFT_VLOG << ss.str();
        r->_update_last_rpc_send_timestamp(rpc_send_time);
        // prev_log_index and prev_log_term doesn't match
        r->_reset_next_index();
        if (response->last_log_index() + 1 < r->_next_index) {
            BRAFT_VLOG << "Group " << r->_options.group_id
                       << " last_log_index at peer=" << r->_options.peer_id
                       << " is " << response->last_log_index();
            // The peer contains less logs than leader
            r->_next_index = response->last_log_index() + 1;
        } else {
            // The peer contains logs from old term which should be truncated,
            // decrease _last_log_at_peer by one to test the right index to keep
            if (BAIDU_LIKELY(r->_next_index > 1)) {
                BRAFT_VLOG << "Group " << r->_options.group_id
                           << " log_index=" << r->_next_index << " mismatch";
                --r->_next_index;
            } else {
                LOG(ERROR) << "Group " << r->_options.group_id
                           << " peer=" << r->_options.peer_id
                           << " declares that log at index=0 doesn't match,"
                              " which is not supposed to happen";
            }
        }
        // dummy_id is unlock in _send_heartbeat
        r->_send_empty_entries(false);
        return;
    }

    ss << " success";
    BRAFT_VLOG << ss.str();

    if (response->term() != r->_options.term) {
        LOG(ERROR) << "Group " << r->_options.group_id
                   << " fail, response term " << response->term()
                   << " mismatch, expect term " << r->_options.term;
        r->_reset_next_index();
        CHECK_EQ(0, bthread_id_unlock(r->_id)) << "Fail to unlock " << r->_id;
        return;
    }
    r->_update_last_rpc_send_timestamp(rpc_send_time);
    const int entries_size = request->entries_size();
    const int64_t rpc_last_log_index = request->prev_log_index() + entries_size;
    BRAFT_VLOG_IF(entries_size > 0) << "Group " << r->_options.group_id
                                    << " replicated logs in ["
                                    << min_flying_index << ", "
                                    << rpc_last_log_index
                                    << "] to peer " << r->_options.peer_id;
    if (entries_size > 0) {
        r->_options.ballot_box->commit_at(
                min_flying_index, rpc_last_log_index,
                r->_options.peer_id);
        int64_t rpc_latency_us = cntl->latency_us();
        if (FLAGS_raft_trace_append_entry_latency &&
            rpc_latency_us > FLAGS_raft_append_entry_high_lat_us) {
            LOG(WARNING) << "append entry rpc latency us " << rpc_latency_us
                         << " greater than "
                         << FLAGS_raft_append_entry_high_lat_us
                         << " Group " << r->_options.group_id
                         << " to peer  " << r->_options.peer_id
                         << " request entry size " << entries_size
                         << " request data size "
                         <<  cntl->request_attachment().size();
        }
        g_send_entries_latency << cntl->latency_us();
        if (cntl->request_attachment().size() > 0) {
            g_normalized_send_entries_latency <<
                cntl->latency_us() * 1024 / cntl->request_attachment().size();
        }
    }
    // A rpc is marked as success, means all request before it are success,
    // erase them sequentially.
    while (!r->_append_entries_in_fly.empty() &&
           r->_append_entries_in_fly.front().log_index <= rpc_first_index) {
        r->_flying_append_entries_size -= r->_append_entries_in_fly.front().entries_size;
        r->_append_entries_in_fly.pop_front();
    }
    r->_has_succeeded = true;
    r->_notify_on_caught_up(0, false);
    ...
    r->_send_entries();
    return;
}
```

```cpp
void Replicator::_notify_on_caught_up(int error_code, bool before_destroy) {
    ...
    Closure* saved_catchup_closure = _catchup_closure;
    _catchup_closure = NULL;
    return run_closure_in_bthread(saved_catchup_closure);
}
```

```cpp
void NodeImpl::on_caughtup(const PeerId& peer, int64_t term,
                           int64_t version, const butil::Status& st) {
    ...
    if (st.ok()) {  // Caught up successfully
        _conf_ctx.on_caughtup(version, peer, true);
        return;
    }
    ...
}
```

```cpp
void NodeImpl::ConfigurationCtx::on_caughtup(
        int64_t version, const PeerId& peer_id, bool succ) {

    ...
    if (succ) {
        _adding_peers.erase(peer_id);
        if (_adding_peers.empty()) {
            return next_stage();
        }
        return;
    }
    ...
}
```

阶段二：联合共识
===

```cpp
void NodeImpl::unsafe_apply_configuration(const Configuration& new_conf,
                                          const Configuration* old_conf,
                                          bool leader_start) {
    LogEntry* entry = new LogEntry();
    entry->AddRef();
    entry->id.term = _current_term;
    entry->type = ENTRY_TYPE_CONFIGURATION;
    entry->peers = new std::vector<PeerId>;
    new_conf.list_peers(entry->peers);
    if (old_conf) {
        entry->old_peers = new std::vector<PeerId>;
        old_conf->list_peers(entry->old_peers);
    }
    ...
    _log_manager->append_entries(&entries,
                                 new LeaderStableClosure(
                                        NodeId(_group_id, _server_id),
                                        1u, _ballot_box));
    _log_manager->check_and_set_configuration(&_conf);
}
```

```cpp
int BallotBox::append_pending_task(const Configuration& conf, const Configuration* old_conf,
                                   Closure* closure) {
    Ballot bl;
    if (bl.init(conf, old_conf) != 0) {
        CHECK(false) << "Fail to init ballot";
        return -1;
    }

    BAIDU_SCOPED_LOCK(_mutex);
    CHECK(_pending_index > 0);
    _pending_meta_queue.push_back(Ballot());
    _pending_meta_queue.back().swap(bl);
    _closure_queue->append_pending_closure(closure);
    return 0;
}
```

```cpp
int Ballot::init(const Configuration& conf, const Configuration* old_conf) {
    _peers.clear();
    _old_peers.clear();
    _quorum = 0;
    _old_quorum = 0;

    _peers.reserve(conf.size());
    for (Configuration::const_iterator
            iter = conf.begin(); iter != conf.end(); ++iter) {
        _peers.push_back(*iter);
    }
    _quorum = _peers.size() / 2 + 1;
    if (!old_conf) {
        return 0;
    }
    _old_peers.reserve(old_conf->size());
    for (Configuration::const_iterator
            iter = old_conf->begin(); iter != old_conf->end(); ++iter) {
        _old_peers.push_back(*iter);
    }
    _old_quorum = _old_peers.size() / 2 + 1;
    return 0;
}
```

```cpp
bool LogManager::check_and_set_configuration(ConfigurationEntry* current) {
    if (current == NULL) {
        CHECK(false) << "current should not be NULL";
        return false;
    }
    BAIDU_SCOPED_LOCK(_mutex);

    const ConfigurationEntry& last_conf = _config_manager->last_configuration();
    if (current->id != last_conf.id) {
        *current = last_conf;
        return true;
    }
    return false;
}
```

```cpp
bool LogManager::check_and_set_configuration(ConfigurationEntry* current) {
    if (current == NULL) {
        CHECK(false) << "current should not be NULL";
        return false;
    }
    BAIDU_SCOPED_LOCK(_mutex);

    const ConfigurationEntry& last_conf = _config_manager->last_configuration();
    if (current->id != last_conf.id) {
        *current = last_conf;
        return true;
    }
    return false;
}
```


follower 接收到 append_entries
---

```cpp
void NodeImpl::become_leader() {
    CHECK(_state == STATE_CANDIDATE);
    LOG(INFO) << "node " << _group_id << ":" << _server_id
              << " term " << _current_term
              << " become leader of group " << _conf.conf
              << " " << _conf.old_conf;
    // cancel candidate vote timer
    _vote_timer.stop();
    _vote_ctx.reset(this);

    _state = STATE_LEADER;
    _leader_id = _server_id;

    _replicator_group.reset_term(_current_term);
    _follower_lease.reset();
    _leader_lease.on_leader_start(_current_term);

    std::set<PeerId> peers;
    _conf.list_peers(&peers);
    for (std::set<PeerId>::const_iterator
            iter = peers.begin(); iter != peers.end(); ++iter) {
        if (*iter == _server_id) {
            continue;
        }

        BRAFT_VLOG << "node " << _group_id << ":" << _server_id
                   << " term " << _current_term
                   << " add replicator " << *iter;
        //TODO: check return code
        _replicator_group.add_replicator(*iter);
    }

    // init commit manager
    _ballot_box->reset_pending_index(_log_manager->last_log_index() + 1);

    // Register _conf_ctx to reject configuration changing before the first log
    // is committed.
    CHECK(!_conf_ctx.is_busy());
    _conf_ctx.flush(_conf.conf, _conf.old_conf);
    _stepdown_timer.start();
}
```

故障恢复
---

```
void NodeImpl::ConfigurationCtx::flush(const Configuration& conf,
                                       const Configuration& old_conf) {
    CHECK(!is_busy());
    conf.list_peers(&_new_peers);
    if (old_conf.empty()) {
        _stage = STAGE_STABLE;
        _old_peers = _new_peers;
    } else {
        _stage = STAGE_JOINT;
        old_conf.list_peers(&_old_peers);
    }
    _node->unsafe_apply_configuration(conf, old_conf.empty() ? NULL : &old_conf,
                                      true);

}
```

TODO
---

* 重复下发有什么问题？

* 讲一下 replicator 的作用
* 讲一下 replicator 的启动流程
* 每条日志会带一个 bollot 还是 boolt box?
* 什么时候往 new_

* 新节点以什么配置生效，会不会成为主？

```cpp
{1,2,3} => {1,4,5} 2 是主?
```

有 leader 的心跳，不会选主？

* 2PC 怎么保证原子性, 负责变更的 leader 挂掉了?

* 假如新节点错误，会不会阻塞后续的变更？不会，在 catchup 阶段就行不通了
