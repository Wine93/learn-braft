整体概览
===

选举流程
---

1. 节点在选举超时（`election_timeout`）时间内未收到任何心跳而触发选举
2. 向所有节点广播 `PreVote` 请求，若收到大多数赞成票则进行正式选举，否则重新等待选举超时
3. 将自身角色转变为 `Candidate`, 并将自身 `Term` 加一，向所有节点广播 `RequestVote` 请求
4. 在投票超时（`vote_timeout`）时间内若收到足够多的选票则成为 `Leader`，若有收到更高 `Term` 的响应则转变为  `Follower` 并重复步骤 1；否则等待投票超时后转变为 `Follower` 并重复步骤 2
5. 成为 Leader
    * 5.1 将自身角色转变为 `Leader`
    * 5.2 对所有 `Follower` 定期广播心跳
    * 5.3 通过提交一条本任期的配置日志来提交上一任期的日志，并回放这些日志来恢复状态机
    * 5.4 回调用户状态机的 `on_leader_start`
6. 至此，Leader 可以正式对外服务

上述流程可分为 PreVote (1-2)、RequestVote (3-4)、成为 Leader（5-6）这三个阶段

投票规则
---

在同一任期内，节点发出的 `PreVote` 和 `RequestVote` 的请求是一样的，区别在于`PreVote` 中的 `Term` 自身的 `Term+1`，而

节点对于 `RequestVote` 请求投赞成票需要同时满足以下 3 个条件：

* Term: 请求中的 `Term` 要大于或等于当前节点的 `Term`
* LastLogId: 请求中的 `LastLogId` 要大于或等于当前节点的 `LastLogId`<sup>[1]</sup>
* votedFor:

唯一的区别在于：

* 节点可以对多个

* `RequestVote` 会记录 `votedFor`，确保在同一个任期内只会给一个候选人投票，而 `PreVote` 则可以同时投票给多个候选人，只要其满足以上 2 个条件 // 补充不会投给其他人
* `RequestVote` 若发现请求中的 `Term` 比自身的大，会 `step_down` 成 Follower，而 `PreVote` 则不会，这点可以确保不会在 Pre-Vote 打断当前 Leader

从以上差异可以看出，`PreVote` 更像是一次预检，检测其连通性和合法性，并没有实际的动作。

> [1] LogId 的比较
>
> LogId 由 Log 的 Term 和 Index 组成，对于 2 个 LogId 来说：
> * 若 `a.Term == b.Term`，则 `a == b`
> * 若 `(a.Term > b.Term) || (a.Term == b.Term && a.Index > b.Index)`，则 `a > b`

一些关键点
---

* `PreVote` 请求中的 Term
* `<currentTerm, votedFor>` 会持久化，这是确保在同一个 Term 内只会产生一个 Leader 的关键
* braft 中成为 Leader 后提交本任期内的第一条日志是配置日志，并非 `NO-OP`
* `CommitIndex` 并不会持久化，Leader 在上述流程中的 5.3 中确认，Follower 则在之后的心跳中由 Leader 传递，只有确认了 `CommitIndex` 后才能开始回放日志

相关 RPC
---

```proto
message RequestVoteRequest {
    required string group_id = 1;
    required string server_id = 2;
    required string peer_id = 3;
    required int64 term = 4;
    required int64 last_log_term = 5;
    required int64 last_log_index = 6;
    optional TermLeader disrupted_leader = 7;
};

message RequestVoteResponse {
    required int64 term = 1;
    required bool granted = 2;
    optional bool disrupted = 3;
    optional int64 previous_term = 4;
    optional bool rejected_by_lease = 5;
};

service RaftService {
    rpc pre_vote(RequestVoteRequest) returns (RequestVoteResponse);
    rpc request_vote(RequestVoteRequest) returns (RequestVoteResponse);
    ...
}
```


阶段一：PreVote
===

![Pre-Vote](image/pre_vote.svg)

触发投票
---

节点在初始化就会启动选举定时器：

```cpp
int NodeImpl::init(const NodeOptions& options) {
    ...
    // 只有当前节点的集群列表不为空，才会调用 step_down 启动选举定时器
    if (!_conf.empty()) {
        step_down(_current_term, false, butil::Status::OK());
    }
    ...
}

void NodeImpl::step_down(const int64_t term, bool wakeup_a_candidate,
                         const butil::Status& status) {
    ...
    _election_timer.start();
}
```

待定时器超时后就会调用 `pre_vote` 进行预投票：

```cpp
// 定时器超时的 handler
void ElectionTimer::run() {
    _node->handle_election_timeout();
}

void NodeImpl::handle_election_timeout() {
    ...
    reset_leader_id(empty_id, status);

    return pre_vote(&lck, triggered);
    // Don't touch any thing of *this ever after
}
```

发送请求
---

在 `pre_vote` 函数中会对所有节点发送 `PreVote` 请求，并设置 RPC 响应的回调函数为 `OnPreVoteRPCDone`， 最后 调用 `grant_slef` 给自己投一票，之后就进入等待：

```cpp
void NodeImpl::pre_vote(std::unique_lock<raft_mutex_t>* lck, bool triggered) {

    const LogId last_log_id = _log_manager->last_log_id(true);

    _pre_vote_ctx.init(this, triggered);
    std::set<PeerId> peers;
    _conf.list_peers(&peers);

    for (std::set<PeerId>::const_iterator
            iter = peers.begin(); iter != peers.end(); ++iter) {
        ...
        OnPreVoteRPCDone* done = new OnPreVoteRPCDone(
                *iter, _current_term, _pre_vote_ctx.version(), this);
        ...
        done->request.set_term(_current_term + 1); // next term
        done->request.set_last_log_index(last_log_id.index);
        done->request.set_last_log_term(last_log_id.term);

        RaftService_Stub stub(&channel);
        stub.pre_vote(&done->cntl, &done->request, &done->response, done);
    }
    grant_self(&_pre_vote_ctx, lck);
}
```

处理请求
---

其他节点在收到 `PreVote` 请求后会调用 `handle_pre_vote_request` 处理请求：

```cpp
int NodeImpl::handle_pre_vote_request(const RequestVoteRequest* request,
                                      RequestVoteResponse* response) {
    ...
    do {
        // (1) 判断 Term
        if (request->term() < _current_term) {
            ...
            break;
        }

        // (2) 判断 LastLogId
        ...
        LogId last_log_id = _log_manager->last_log_id(true);
        ...
        bool grantable = (LogId(request->last_log_index(), request->last_log_term())
                        >= last_log_id);
        if (grantable) {
            granted = (votable_time == 0);
        }
        ...
    } while (0);

    // (3) 设置响应
    ...
    response->set_term(_current_term);
    response->set_granted(granted);  //
    ...

    return 0;
}

```

处理响应
---

在收到其他节点的 `PreVote` 响应后，会回调之前设置的 callback `OnPreVoteRPCDone->Run()`，在 callback 中会调用 `handle_pre_vote_response` 处理 `PreVote` 响应：

```cpp
struct OnPreVoteRPCDone : public google::protobuf::Closure {
    ...
    void Run() {
            if (cntl.ErrorCode() != 0) {
                ...
                break;
            }
            node->handle_pre_vote_response(peer, term, ctx_version, response);
    }
    ...
};
```

处理 `PreVote` 响应：
```cpp

```

投票失败
---


阶段二：`RequestVote`
===

![alt text](image/vote.svg)

发送请求
---

当 PreVote 阶段获得大多数节点的支持后，将调用 `elect_self` 正式进 *RequestVote* 阶段。在 `elect_self` 会将角色转变为 Candidte，并加自身的 Term + 1，向所有的节点发送 `RequestVote` 请求，最后给自己投一票后，等待其他节点的 `RequestVote` 响应：

```cpp
void NodeImpl::elect_self(std::unique_lock<raft_mutex_t>* lck,
                          bool old_leader_stepped_down) {
    ...

    _state = STATE_CANDIDATE;  //
    _current_term++;           // 将自身的 Term+1
    _voted_id = _server_id;    // 记录 votedFor 投给自己

    ...
    // 启动投票超时器：如果在 vote_timeout 未得到足够多的选票，则变为 Follower 重新进行 PreVote
    _vote_timer.start();

    const LogId last_log_id = _log_manager->last_log_id(true);

    _vote_ctx.set_last_log_id(last_log_id);

    std::set<PeerId> peers;
    _conf.list_peers(&peers);
    request_peers_to_vote(peers, _vote_ctx.disrupted_leader());

    // 持久化 votedFor
    status = _meta_storage->
                    set_term_and_votedfor(_current_term, _server_id, _v_group_id);
    grant_self(&_vote_ctx, lck);
}
```

```cpp
void NodeImpl::request_peers_to_vote(const std::set<PeerId>& peers,
                                     const DisruptedLeader& disrupted_leader) {
    for (std::set<PeerId>::const_iterator
        iter = peers.begin(); iter != peers.end(); ++iter) {
        ...
        OnRequestVoteRPCDone* done =
            new OnRequestVoteRPCDone(*iter, _current_term, _vote_ctx.version(), this);
        ...
        done->request.set_term(_current_term);
        done->request.set_last_log_index(_vote_ctx.last_log_id().index);
        done->request.set_last_log_term(_vote_ctx.last_log_id().term);

        RaftService_Stub stub(&channel);
        stub.request_vote(&done->cntl, &done->request, &done->response, done);
    }
}
```

处理请求
---

节点在收到 `RequestVote` 请求后，会调用 `handle_request_vote_request`

```cpp
int NodeImpl::handle_request_vote_request(const RequestVoteRequest* request,
                                          RequestVoteResponse* response) {
    ...
    PeerId disrupted_leader_id;
    if (_state == STATE_FOLLOWER &&
            request->has_disrupted_leader() &&
            _current_term == request->disrupted_leader().term() &&
            0 == disrupted_leader_id.parse(request->disrupted_leader().peer_id()) &&
            _leader_id == disrupted_leader_id) {
        // The candidate has already disrupted the old leader, we
        // can expire the lease safely.
        _follower_lease.expire();
    }

    bool disrupted = false;
    int64_t previous_term = _current_term;
    bool rejected_by_lease = false;
    do {
        // ignore older term
        if (request->term() < _current_term) {
            // ignore older term
            LOG(INFO) << "node " << _group_id << ":" << _server_id
                      << " ignore RequestVote from " << request->server_id()
                      << " in term " << request->term()
                      << " current_term " << _current_term;
            break;
        }

        // get last_log_id outof node mutex
        lck.unlock();
        LogId last_log_id = _log_manager->last_log_id(true);
        lck.lock();


        bool log_is_ok = (LogId(request->last_log_index(), request->last_log_term())
                          >= last_log_id);
        int64_t votable_time = _follower_lease.votable_time_from_now();



        // if the vote is rejected by lease, tell the candidate
        if (votable_time > 0) {  // 大于 0 代表还不可以投票
            rejected_by_lease = log_is_ok;
            break;
        }

        // increase current term, change state to follower
        if (request->term() > _current_term) {
            ...
            step_down(request->term(), false, status);
        }

        if (log_is_ok && _voted_id.is_empty()) {
            ...
            step_down(request->term(), false, status);
            _voted_id = candidate_id;  // 记录 votedFor
            status = _meta_storage->
                    set_term_and_votedfor(_current_term, candidate_id, _v_group_id);
        }
    } while (0);

    response->set_disrupted(disrupted);
    response->set_previous_term(previous_term);
    response->set_term(_current_term);
    response->set_granted(request->term() == _current_term && _voted_id == candidate_id);
    response->set_rejected_by_lease(rejected_by_lease);
    return 0;
}
```

处理响应
---

```cpp
struct OnRequestVoteRPCDone : public google::protobuf::Closure {
    ...
    void Run() {
            if (cntl.ErrorCode() != 0) {
                ...
                break;
            }
            node->handle_request_vote_response(peer, term, ctx_version, response);
    }
    ...
};
```

```cpp
void NodeImpl::handle_request_vote_response(const PeerId& peer_id, const int64_t term,
                                            const int64_t ctx_version,
                                            const RequestVoteResponse& response) {
    ...
    // (1) 发现有比自己 Term 高的节点，则 step_down 成 Follower
    if (response.term() > _current_term) {
        ...
        step_down(response.term(), false, status);
        return;
    }
    ...
    //
    if (!response.granted() && !response.rejected_by_lease()) {
        return;
    }

    if (response.granted()) {
        _vote_ctx.grant(peer_id);
        if (peer_id == _follower_lease.last_leader()) {
            _vote_ctx.grant(_server_id);
            _vote_ctx.stop_grant_self_timer(this);
        }
        if (_vote_ctx.granted()) {
            return become_leader();
        }
    } else {
        // If the follower rejected the vote because of lease, reserve it, and
        // the candidate will try again after it disrupt the old leader.
        _vote_ctx.reserve(peer_id);
    }
    retry_vote_on_reserved_peers();  // 这个有啥作用?
}
```


投票超时
---

```cpp
void VoteTimer::run() {
    _node->handle_vote_timeout();
}

void NodeImpl::handle_vote_timeout() {
    ...
    step_down(_current_term, false, status);
    pre_vote(&lck, false);
    ...
}
```

阶段三：成为 *Leader*
===

```cpp
// in lock
void NodeImpl::become_leader() {
    ...
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
        ...
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

创建 Replicator
---

节点在成为 Leader 后会为每个 Follower 创建对应 `Replicator`，每个 `Replicator` 都是单独的 `bthread`，它主要有以下 3 个作用：

* 记录 Follower 的一些状态，包括 `next_index`
* 作为 RPC Client，所有从 Leader 发往 Follower 的 RPC 请求都会通过它，包括心跳、`AppendEntriesRequest`、`InstallSnapshotRequest`
* 最重要的就是复制日志，Replicator 默认在后台等待；当 Leader 通过 `LogManager` 追加日志时，就会唤醒 Replicator 进行发送日志，发送完了继续后台等待新日志的到来，整个过来是个流水线式的实现，没有任何阻塞。

```cpp
int ReplicatorGroup::add_replicator(const PeerId& peer) {
    CHECK_NE(0, _common_options.term);
    if (_rmap.find(peer) != _rmap.end()) {
        return 0;
    }
    ReplicatorOptions options = _common_options;
    options.peer_id = peer;
    options.replicator_status = new ReplicatorStatus;
    ReplicatorId rid;
    if (Replicator::start(options, &rid) != 0) {
        LOG(ERROR) << "Group " << options.group_id
                   << " Fail to start replicator to peer=" << peer;
        delete options.replicator_status;
        return -1;
    }
    _rmap[peer] = { rid, options.replicator_status };
    return 0;
}
```

```cpp
int Replicator::start(const ReplicatorOptions& options, ReplicatorId *id) {
    if (options.log_manager == NULL || options.ballot_box == NULL
            || options.node == NULL) {
        LOG(ERROR) << "Invalid arguments, group " << options.group_id;
        return -1;
    }
    Replicator* r = new Replicator();
    brpc::ChannelOptions channel_opt;
    channel_opt.connect_timeout_ms = FLAGS_raft_rpc_channel_connect_timeout_ms;
    channel_opt.timeout_ms = -1; // We don't need RPC timeout
    if (r->_sending_channel.Init(options.peer_id.addr, &channel_opt) != 0) {
        LOG(ERROR) << "Fail to init sending channel"
                   << ", group " << options.group_id;
        delete r;
        return -1;
    }

    // bind lifecycle with node, AddRef
    // Replicator stop is async
    options.node->AddRef();
    options.replicator_status->AddRef();
    r->_options = options;
    r->_next_index = r->_options.log_manager->last_log_index() + 1;
    if (bthread_id_create(&r->_id, r, _on_error) != 0) {
        LOG(ERROR) << "Fail to create bthread_id"
                   << ", group " << options.group_id;
        delete r;
        return -1;
    }


    bthread_id_lock(r->_id, NULL);
    if (id) {
        *id = r->_id.value;
    }
    LOG(INFO) << "Replicator=" << r->_id << "@" << r->_options.peer_id << " is started"
              << ", group " << r->_options.group_id;
    r->_catchup_closure = NULL;
    r->_update_last_rpc_send_timestamp(butil::monotonic_time_ms());
    r->_start_heartbeat_timer(butil::gettimeofday_us());
    // Note: r->_id is unlock in _send_empty_entries, don't touch r ever after
    r->_send_empty_entries(false);
    return 0;
}
```

发送心跳
---
```cpp
static inline int heartbeat_timeout(int election_timeout) {
    if (FLAGS_raft_election_heartbeat_factor <= 0){
        LOG(WARNING) << "raft_election_heartbeat_factor flag must be greater than 1"
                     << ", but get "<< FLAGS_raft_election_heartbeat_factor
                     << ", it will be set to default value 10.";
        FLAGS_raft_election_heartbeat_factor = 10;
    }
    return std::max(election_timeout / FLAGS_raft_election_heartbeat_factor, 10);
}

void Replicator::_on_timedout(void* arg) {
    bthread_id_t id = { (uint64_t)arg };
    bthread_id_error(id, ETIMEDOUT);
}

void Replicator::_start_heartbeat_timer(long start_time_us) {
    const timespec due_time = butil::milliseconds_from(
            butil::microseconds_to_timespec(start_time_us),
            *_options.dynamic_heartbeat_timeout_ms);
    if (bthread_timer_add(&_heartbeat_timer, due_time,
                       _on_timedout, (void*)_id.value) != 0) {
        _on_timedout((void*)_id.value);
    }
}

void* Replicator::_send_heartbeat(void* arg) {
    Replicator* r = NULL;
    bthread_id_t id = { (uint64_t)arg };
    if (bthread_id_lock(id, (void**)&r) != 0) {
        // This replicator is stopped
        return NULL;
    }
    // id is unlock in _send_empty_entries;
    r->_send_empty_entries(true);
    return NULL;
}

int Replicator::_on_error(bthread_id_t id, void* arg, int error_code) {
    Replicator* r = (Replicator*)arg;
    if (error_code == ESTOP) {
        brpc::StartCancel(r->_install_snapshot_in_fly);
        brpc::StartCancel(r->_heartbeat_in_fly);
        brpc::StartCancel(r->_timeout_now_in_fly);
        r->_cancel_append_entries_rpcs();
        bthread_timer_del(r->_heartbeat_timer);
        r->_options.log_manager->remove_waiter(r->_wait_id);
        r->_notify_on_caught_up(error_code, true);
        r->_wait_id = 0;
        LOG(INFO) << "Group " << r->_options.group_id
                  << " Replicator=" << id << " is going to quit";
        r->_destroy();
        return 0;
    } else if (error_code == ETIMEDOUT) {
        // This error is issued in the TimerThread, start a new bthread to avoid
        // blocking the caller.
        // Unlock id to remove the context-switch out of the critical section
        CHECK_EQ(0, bthread_id_unlock(id)) << "Fail to unlock" << id;
        bthread_t tid;
        if (bthread_start_urgent(&tid, NULL, _send_heartbeat,
                                 reinterpret_cast<void*>(id.value)) != 0) {
            PLOG(ERROR) << "Fail to start bthread";
            _send_heartbeat(reinterpret_cast<void*>(id.value));
        }
        return 0;
    } else {
        CHECK(false) << "Group " << r->_options.group_id
                     << " Unknown error_code=" << error_code;
        CHECK_EQ(0, bthread_id_unlock(id)) << "Fail to unlock " << id;
        return -1;
    }
}
```


确定 next_index
---

```cpp
void Replicator::_send_empty_entries(bool is_heartbeat) {
    std::unique_ptr<brpc::Controller> cntl(new brpc::Controller);
    std::unique_ptr<AppendEntriesRequest> request(new AppendEntriesRequest);
    std::unique_ptr<AppendEntriesResponse> response(new AppendEntriesResponse);
    if (_fill_common_fields(
                request.get(), _next_index - 1, is_heartbeat) != 0) {
        CHECK(!is_heartbeat);
        // _id is unlock in _install_snapshot
        return _install_snapshot();
    }
    if (is_heartbeat) {
        _heartbeat_in_fly = cntl->call_id();
        _heartbeat_counter++;
        // set RPC timeout for heartbeat, how long should timeout be is waiting to be optimized.
        cntl->set_timeout_ms(*_options.election_timeout_ms / 2);
    } else {
        _st.st = APPENDING_ENTRIES;
        _st.first_log_index = _next_index;
        _st.last_log_index = _next_index - 1;
        CHECK(_append_entries_in_fly.empty());
        CHECK_EQ(_flying_append_entries_size, 0);
        _append_entries_in_fly.push_back(FlyingAppendEntriesRpc(_next_index, 0, cntl->call_id()));
        _append_entries_counter++;
    }

    BRAFT_VLOG << "node " << _options.group_id << ":" << _options.server_id
        << " send HeartbeatRequest to " << _options.peer_id
        << " term " << _options.term
        << " prev_log_index " << request->prev_log_index()
        << " last_committed_index " << request->committed_index();

    google::protobuf::Closure* done = brpc::NewCallback(
                is_heartbeat ? _on_heartbeat_returned : _on_rpc_returned,
                _id.value, cntl.get(), request.get(), response.get(),
                butil::monotonic_time_ms());

    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
    CHECK_EQ(0, bthread_id_unlock(_id)) << "Fail to unlock " << _id;
}
```

```cpp
int Replicator::_fill_common_fields(AppendEntriesRequest* request,
                                    int64_t prev_log_index,  // leader 中记录该 follower 的 next_log_id
                                    bool is_heartbeat) {
    const int64_t prev_log_term = _options.log_manager->get_term(prev_log_index);
    if (prev_log_term == 0 && prev_log_index != 0) {  // 查不到对应日志的 term，代表已经被快照压缩了
        if (!is_heartbeat) {
            CHECK_LT(prev_log_index, _options.log_manager->first_log_index());
            BRAFT_VLOG << "Group " << _options.group_id
                       << " log_index=" << prev_log_index << " was compacted";
            return -1;
        } else {
            // The log at prev_log_index has been compacted, which indicates
            // we are or are going to install snapshot to the follower. So we let
            // both prev_log_index and prev_log_term be 0 in the heartbeat
            // request so that follower would do nothing besides updating its
            // leader timestamp.
            prev_log_index = 0;
        }
    }
    request->set_term(_options.term);
    request->set_group_id(_options.group_id);
    request->set_server_id(_options.server_id.to_string());
    request->set_peer_id(_options.peer_id.to_string());
    request->set_prev_log_index(prev_log_index);
    request->set_prev_log_term(prev_log_term);
    request->set_committed_index(_options.ballot_box->last_committed_index());
    return 0;
}
```

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
    // dummy_id is unlock in _send_entries
    if (r->_timeout_now_index > 0 && r->_timeout_now_index < r->_min_flying_index()) {
        r->_send_timeout_now(false, false);
    }
    r->_send_entries();
    return;
}
```

```cpp
void NodeImpl::handle_append_entries_request(brpc::Controller* cntl,
                                             const AppendEntriesRequest* request,
                                             AppendEntriesResponse* response,
                                             google::protobuf::Closure* done,
                                             bool from_append_entries_cache) {
    std::vector<LogEntry*> entries;
    entries.reserve(request->entries_size());
    brpc::ClosureGuard done_guard(done);
    std::unique_lock<raft_mutex_t> lck(_mutex);

    // pre set term, to avoid get term in lock
    response->set_term(_current_term);

    if (!is_active_state(_state)) {
        const int64_t saved_current_term = _current_term;
        const State saved_state = _state;
        lck.unlock();
        LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " is not in active state " << "current_term " << saved_current_term
                     << " state " << state2str(saved_state);
        cntl->SetFailed(EINVAL, "node %s:%s is not in active state, state %s",
                _group_id.c_str(), _server_id.to_string().c_str(), state2str(saved_state));
        return;
    }

    PeerId server_id;
    if (0 != server_id.parse(request->server_id())) {
        lck.unlock();
        LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " received AppendEntries from " << request->server_id()
                     << " server_id bad format";
        cntl->SetFailed(brpc::EREQUEST,
                        "Fail to parse server_id `%s'",
                        request->server_id().c_str());
        return;
    }

    // check stale term
    if (request->term() < _current_term) {
        const int64_t saved_current_term = _current_term;
        lck.unlock();
        LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " ignore stale AppendEntries from " << request->server_id()
                     << " in term " << request->term()
                     << " current_term " << saved_current_term;
        response->set_success(false);
        response->set_term(saved_current_term);
        return;
    }

    // check term and state to step down
    check_step_down(request->term(), server_id);

    if (server_id != _leader_id) {
        LOG(ERROR) << "Another peer " << _group_id << ":" << server_id
                   << " declares that it is the leader at term=" << _current_term
                   << " which was occupied by leader=" << _leader_id;
        // Increase the term by 1 and make both leaders step down to minimize the
        // loss of split brain
        butil::Status status;
        status.set_error(ELEADERCONFLICT, "More than one leader in the same term.");
        step_down(request->term() + 1, false, status);
        response->set_success(false);
        response->set_term(request->term() + 1);
        return;
    }

    if (!from_append_entries_cache) {
        // Requests from cache already updated timestamp
        _follower_lease.renew(_leader_id);
    }

    if (request->entries_size() > 0 &&
            (_snapshot_executor
                && _snapshot_executor->is_installing_snapshot())) {
        LOG(WARNING) << "node " << _group_id << ":" << _server_id
                     << " received append entries while installing snapshot";
        cntl->SetFailed(EBUSY, "Is installing snapshot");
        return;
    }

    const int64_t prev_log_index = request->prev_log_index();
    const int64_t prev_log_term = request->prev_log_term();
    const int64_t local_prev_log_term = _log_manager->get_term(prev_log_index);
    if (local_prev_log_term != prev_log_term) {
        int64_t last_index = _log_manager->last_log_index();
        int64_t saved_term = request->term();
        int     saved_entries_size = request->entries_size();
        std::string rpc_server_id = request->server_id();
        if (!from_append_entries_cache &&
            handle_out_of_order_append_entries(
                    cntl, request, response, done, last_index)) {
            // It's not safe to touch cntl/request/response/done after this point,
            // since the ownership is tranfered to the cache.
            lck.unlock();
            done_guard.release();
            LOG(WARNING) << "node " << _group_id << ":" << _server_id
                         << " cache out-of-order AppendEntries from "
                         << rpc_server_id
                         << " in term " << saved_term
                         << " prev_log_index " << prev_log_index
                         << " prev_log_term " << prev_log_term
                         << " local_prev_log_term " << local_prev_log_term
                         << " last_log_index " << last_index
                         << " entries_size " << saved_entries_size;
            return;
        }

        response->set_success(false);
        response->set_term(_current_term);
        response->set_last_log_index(last_index);  // 响应当前 follower 的最后一条日志的 index
        lck.unlock();
        if (local_prev_log_term != 0) {
            LOG(WARNING) << "node " << _group_id << ":" << _server_id
                         << " reject term_unmatched AppendEntries from "
                         << request->server_id()
                         << " in term " << request->term()
                         << " prev_log_index " << request->prev_log_index()
                         << " prev_log_term " << request->prev_log_term()
                         << " local_prev_log_term " << local_prev_log_term
                         << " last_log_index " << last_index
                         << " entries_size " << request->entries_size()
                         << " from_append_entries_cache: " << from_append_entries_cache;
        }
        return;
    }

    if (request->entries_size() == 0) {  // 响应 send_empty_entries 或心跳
        response->set_success(true);
        response->set_term(_current_term);
        response->set_last_log_index(_log_manager->last_log_index());
        response->set_readonly(_node_readonly);
        lck.unlock();
        // see the comments at FollowerStableClosure::run()
        _ballot_box->set_last_committed_index(
                std::min(request->committed_index(),
                         prev_log_index));
        return;
    }

    // Parse request
    butil::IOBuf data_buf;
    data_buf.swap(cntl->request_attachment());
    int64_t index = prev_log_index;
    for (int i = 0; i < request->entries_size(); i++) {
        index++;
        const EntryMeta& entry = request->entries(i);
        if (entry.type() != ENTRY_TYPE_UNKNOWN) {
            LogEntry* log_entry = new LogEntry();
            log_entry->AddRef();
            log_entry->id.term = entry.term();
            log_entry->id.index = index;
            log_entry->type = (EntryType)entry.type();
            if (entry.peers_size() > 0) {  // 配置日志，C{new}
                log_entry->peers = new std::vector<PeerId>;
                for (int i = 0; i < entry.peers_size(); i++) {
                    log_entry->peers->push_back(entry.peers(i));
                }
                CHECK_EQ(log_entry->type, ENTRY_TYPE_CONFIGURATION);
                if (entry.old_peers_size() > 0) {  // 配置变更的日志, C{old,new}
                    log_entry->old_peers = new std::vector<PeerId>;
                    for (int i = 0; i < entry.old_peers_size(); i++) {
                        log_entry->old_peers->push_back(entry.old_peers(i));
                    }
                }
            } else {
                CHECK_NE(entry.type(), ENTRY_TYPE_CONFIGURATION);
            }
            if (entry.has_data_len()) {
                int len = entry.data_len();
                data_buf.cutn(&log_entry->data, len);
            }
            entries.push_back(log_entry);
        }
    }

    // check out-of-order cache
    check_append_entries_cache(index);

    FollowerStableClosure* c = new FollowerStableClosure(
            cntl, request, response, done_guard.release(),
            this, _current_term);
    _log_manager->append_entries(&entries, c);

    // update configuration after _log_manager updated its memory status
    _log_manager->check_and_set_configuration(&_conf);
}
```






发送 no-op
---

```cpp
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

void NodeImpl::unsafe_apply_configuration(const Configuration& new_conf,
                                          const Configuration* old_conf,
                                          bool leader_start) {
    CHECK(_conf_ctx.is_busy());
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
    ConfigurationChangeDone* configuration_change_done =
            new ConfigurationChangeDone(this, _current_term, leader_start, _leader_lease.lease_epoch());
    // Use the new_conf to deal the quorum of this very log
    _ballot_box->append_pending_task(new_conf, old_conf, configuration_change_done);

    std::vector<LogEntry*> entries;
    entries.push_back(entry);
    _log_manager->append_entries(&entries,
                                 new LeaderStableClosure(
                                        NodeId(_group_id, _server_id),
                                        1u, _ballot_box));
    _log_manager->check_and_set_configuration(&_conf);
}
```

下面涉及到日志复制的流程，我们在这里只列出关键逻辑，之后会在日志复制章节详细讲解。

```cpp
// 将 index 在 [fist_log_index, last_log_index] 之间的日志的投票数加一
int BallotBox::commit_at(
        int64_t first_log_index, int64_t last_log_index, const PeerId& peer) {
    // FIXME(chenzhangyi01): The cricital section is unacceptable because it
    // blocks all the other Replicators and LogManagers
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (_pending_index == 0) {
        return EINVAL;
    }
    if (last_log_index < _pending_index) {
        return 0;
    }
    if (last_log_index >= _pending_index + (int64_t)_pending_meta_queue.size()) {
        return ERANGE;
    }

    int64_t last_committed_index = 0;
    const int64_t start_at = std::max(_pending_index, first_log_index);
    Ballot::PosHint pos_hint;
    for (int64_t log_index = start_at; log_index <= last_log_index; ++log_index) {
        Ballot& bl = _pending_meta_queue[log_index - _pending_index];
        pos_hint = bl.grant(peer, pos_hint);
        if (bl.granted()) {
            last_committed_index = log_index;
        }
    }

    if (last_committed_index == 0) {
        return 0;
    }

    // When removing a peer off the raft group which contains even number of
    // peers, the quorum would decrease by 1, e.g. 3 of 4 changes to 2 of 3. In
    // this case, the log after removal may be committed before some previous
    // logs, since we use the new configuration to deal the quorum of the
    // removal request, we think it's safe to commit all the uncommitted
    // previous logs, which is not well proved right now
    // TODO: add vlog when committing previous logs
    for (int64_t index = _pending_index; index <= last_committed_index; ++index) {
        _pending_meta_queue.pop_front();
    }

    _pending_index = last_committed_index + 1;
    _last_committed_index.store(last_committed_index, butil::memory_order_relaxed);
    lck.unlock();
    // The order doesn't matter
    _waiter->on_committed(last_committed_index);
    return 0;
}
```

```cpp
void FSMCaller::do_committed(int64_t committed_index) {
    if (!_error.status().ok()) {
        return;
    }
    int64_t last_applied_index = _last_applied_index.load(
                                        butil::memory_order_relaxed);

    // We can tolerate the disorder of committed_index
    if (last_applied_index >= committed_index) {
        return;
    }
    std::vector<Closure*> closure;
    int64_t first_closure_index = 0;
    CHECK_EQ(0, _closure_queue->pop_closure_until(committed_index, &closure,
                                                  &first_closure_index));

    IteratorImpl iter_impl(_fsm, _log_manager, &closure, first_closure_index,
                 last_applied_index, committed_index, &_applying_index);
    for (; iter_impl.is_good();) {
        if (iter_impl.entry()->type != ENTRY_TYPE_DATA) {
            if (iter_impl.entry()->type == ENTRY_TYPE_CONFIGURATION) {
                if (iter_impl.entry()->old_peers == NULL) {
                    // Joint stage is not supposed to be noticeable by end users.
                    _fsm->on_configuration_committed(
                            Configuration(*iter_impl.entry()->peers),
                            iter_impl.entry()->id.index);
                }
            }
            // For other entries, we have nothing to do besides flush the
            // pending tasks and run this closure to notify the caller that the
            // entries before this one were successfully committed and applied.
            if (iter_impl.done()) {
                iter_impl.done()->Run();
            }
            iter_impl.next();
            continue;
        }
        Iterator iter(&iter_impl);
        _fsm->on_apply(iter);
        LOG_IF(ERROR, iter.valid())
                << "Node " << _node->node_id()
                << " Iterator is still valid, did you return before iterator "
                   " reached the end?";
        // Try move to next in case that we pass the same log twice.
        iter.next();
    }
    if (iter_impl.has_error()) {
        set_error(iter_impl.error());
        iter_impl.run_the_rest_closure_with_error();
    }
    const int64_t last_index = iter_impl.index() - 1;
    const int64_t last_term = _log_manager->get_term(last_index);
    LogId last_applied_id(last_index, last_term);
    _last_applied_index.store(committed_index, butil::memory_order_release);
    _last_applied_term = last_term;
    _log_manager->set_applied_id(last_applied_id);
}
```

回调 on_leader_start
---

```cpp
class ConfigurationChangeDone : public Closure {
public:
    void Run() {
        ...
        // 回调用户的状态机的 on_leader_start，_term 为当前 Leader 的 Term
        if (_leader_start) {
            _node->_options.fsm->on_leader_start(_term);
        }
        ...
    }
    ...
};
```