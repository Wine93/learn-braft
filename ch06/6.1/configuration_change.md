# 配置变更

* 简介下各个变更的简略流程
* 快照启动时这么读取配置，集群配置会打到快照里面吗？

整体流程
===

阶段一：ca



配置变更
===

新节点启动的时候需要为空节点，否则可能需要脑裂
参考 https://github.com/baidu/braft/issues/303

TODO
---

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

```cpp
// Add a new peer into the replicating group which consists of |conf|.
// Returns OK on success, error information otherwise.
butil::Status add_peer(const GroupId& group_id, const Configuration& conf,
                       const PeerId& peer_id, const CliOptions& options);

// Remove a peer from the replicating group which consists of |conf|.
// Returns OK on success, error information otherwise.
butil::Status remove_peer(const GroupId& group_id, const Configuration& conf,
                          const PeerId& peer_id, const CliOptions& options);

// Gracefully change the peers of the replication group.
butil::Status change_peers(const GroupId& group_id, const Configuration& conf,
                           const Configuration& new_peers,
                           const CliOptions& options);

// Transfer the leader of the replication group to the target peer
butil::Status transfer_leader(const GroupId& group_id, const Configuration& conf,
                              const PeerId& peer, const CliOptions& options);

// Reset the peer set of the target peer
butil::Status reset_peer(const GroupId& group_id, const PeerId& peer_id,
                         const Configuration& new_conf,
                         const CliOptions& options);
```

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
```

```cpp
add_pper
remove_peer
reset_peer
```

```cpp
void NodeImpl::add_peer(const PeerId& peer, Closure* done) {
    BAIDU_SCOPED_LOCK(_mutex);
    Configuration new_conf = _conf.conf;
    new_conf.add_peer(peer);
    return unsafe_register_conf_change(_conf.conf, new_conf, done);
}
```


```cpp
void NodeImpl::unsafe_register_conf_change(const Configuration& old_conf,
                                           const Configuration& new_conf,
                                           Closure* done) {
    ...
    if (_conf_ctx.is_busy()) {
        ...
        return;
    }
    ...
    return _conf_ctx.start(old_conf, new_conf, done);
}
```

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
---

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
===

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