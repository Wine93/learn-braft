




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

