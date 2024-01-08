

* 提交任务后，日志后被立马发送到各个 follower 吗？还是会等到下一次心跳

```
void NodeImpl::apply(const Task& task) {
    _apply_queue->execute(m, &bthread::TASK_OPTIONS_INPLACE, NULL)
}
```

```proto
message AppendEntriesRequest {
    required string group_id = 1;
    required string server_id = 2;
    required string peer_id = 3;
    required int64 term = 4;
    required int64 prev_log_term = 5;
    required int64 prev_log_index = 6;
    repeated EntryMeta entries = 7;
    required int64 committed_index = 8;
};

message AppendEntriesResponse {
    required int64 term = 1;
    required bool success = 2;
    optional int64 last_log_index = 3;
    optional bool readonly = 4;
};
```

```cpp
int NodeImpl::execute_applying_tasks(void* meta,
                                     bthread::TaskIterator<LogEntryAndClosure>& iter) {
}
```


```cpp
void NodeImpl::apply(LogEntryAndClosure tasks[], size_t size) {

    for (size_t i = 0; i < size; ++i) {

        _ballot_box->append_pending_task(_conf.conf,
                                         _conf.stable() ? NULL : &_conf.old_conf,
                                         tasks[i].done);

    }

    _log_manager->append_entries(&entries,
                                 new LeaderStableClosure(NodeId(_group_id, _server_id),
                                                         entries.size(),
                                                         _ballot_box));

}
```

```cpp
void LogManager::append_entries(std::vector<LogEntry*> *entries,
                                StableClosure* done) {

    int ret = bthread::execution_queue_execute(_disk_queue, done);
    wakeup_all_waiter(lck);
}
```

```cpp
int LogManager::disk_thread(void* meta,
                            bthread::TaskIterator<StableClosure*>& iter) {

    AppendBatcher ab(...);
    for (; iter; ++iter) {
        StableClosure* done = *iter;

        ab.append(done);
        ab.flush();

        done->Run();

    }
}
```

```cpp
class AppendBatcher {

    void flush() {
        _lm->append_to_storage(&_to_append, _last_id, &metric);
    }
};
```

```cpp
void LogManager::append_to_storage(std::vector<LogEntry*>* to_append,
                                   LogId* last_id,
                                   IOMetric* metric) {
    int nappent = _log_storage->append_entries(*to_append, metric);
}
```

```cpp
int SegmentLogStorage::append_entries(const std::vector<LogEntry*>& entries, IOMetric* metric)
    for (size_t i = 0; i < entries.size(); i++) {
        scoped_refptr<Segment> segment = open_segment();
        segment->append(entry);

        _last_log_index.fetch_add(1, butil::memory_order_release);
        last_segment = segment;
    }

    last_segment->sync(_enable_sync);
```

```cpp
void LeaderStableClosure::Run()
    _ballot_box->commit_at(_first_log_index,
                           _first_log_index + _nentries - 1,
                           _node_id.peer_id);
```

```cpp
int BallotBox::commit_at(int64_t first_log_index,
                         int64_t last_log_index,
                         const PeerId& peer) {

    _waiter->on_committed(last_committed_index);
}
```

```
int ReplicatorGroup::add_replicator(const PeerId& peer)
    Replicator::start(options, &rid);

    _rmap[peer] = { rid, options.replicator_status };
```

```cpp
int Replicator::start(const ReplicatorOptions& options, ReplicatorId *id)

    Replicator* r = new Replicator();

    r->_start_heartbeat_timer(butil::gettimeofday_us());
    r->_send_empty_entries(false);
```

```cpp
void Replicator::_send_empty_entries(bool is_heartbeat)

    _fill_common_fields(request.get(), _next_index - 1, is_heartbeat)

    google::protobuf::Closure* done = brpc::NewCallback(
                is_heartbeat ? _on_heartbeat_returned : _on_rpc_returned,
                _id.value, cntl.get(), request.get(), response.get(),
                butil::monotonic_time_ms())

    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
```

```cpp
int Replicator::_fill_common_fields(AppendEntriesRequest* request,
                                    int64_t prev_log_index,
                                    bool is_heartbeat)

    request->set_term(_options.term);
    request->set_group_id(_options.group_id);
    request->set_server_id(_options.server_id.to_string());
    request->set_peer_id(_options.peer_id.to_string());
    request->set_prev_log_index(prev_log_index);
    request->set_prev_log_term(prev_log_term);
    request->set_committed_index(_options.ballot_box->last_committed_index());
    return 0;
```

```
void Replicator::_on_rpc_returned(ReplicatorId id, brpc::Controller* cntl,
                                  AppendEntriesRequest* request,
                                  AppendEntriesResponse* response,
                                  int64_t rpc_send_time)

    r->_send_entries();
```

```cpp
void Replicator::_send_entries()

    google::protobuf::Closure* done = brpc::NewCallback(
                _on_rpc_returned, _id.value, cntl.get(),
                request.get(), response.get(), butil::monotonic_time_ms());
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
    _wait_more_entries();
```

```cpp
void Replicator::_wait_more_entries()
    // 回调 _continue_sending
    _wait_id = _options.log_manager->wait(_next_index - 1,
                                          _continue_sending,
                                          (void*)_id.value);
```

```cpp
int Replicator::_continue_sending(void* arg, int error_code)
    r->_send_entries();
```


follower 接收日志
---

// 问题：会不会 apply 上一个任期的日志？

```cpp
void RaftServiceImpl::append_entries(google::protobuf::RpcController* cntl_base,
                                     const AppendEntriesRequest* request,
                                     AppendEntriesResponse* response,
                                     google::protobuf::Closure* done)

    return node->handle_append_entries_request(cntl, request, response,
                                               done_guard.release());
```

```cpp
void NodeImpl::handle_append_entries_request(brpc::Controller* cntl,
                                             const AppendEntriesRequest* request,
                                             AppendEntriesResponse* response,
                                             google::protobuf::Closure* done,
                                             bool from_append_entries_cache)

    if (request->term() < _current_term) {
        response->set_success(false);
        response->set_term(saved_current_term);
        return;
    }

    const int64_t prev_log_index = request->prev_log_index();
    const int64_t prev_log_term = request->prev_log_term();
    const int64_t local_prev_log_term = _log_manager->get_term(prev_log_index);
    if (local_prev_log_term != prev_log_term) {

        response->set_success(false);
        response->set_term(_current_term);
        response->set_last_log_index(last_index);
        return;
    }

     for (int i = 0; i < request->entries_size(); i++) {
     }

    FollowerStableClosure* c = new FollowerStableClosure(
            cntl, request, response, done_guard.release(),
            this, _current_term);
    _log_manager->append_entries(&entries, c);
```

```cpp
void LogManager::append_entries(std::vector<LogEntry*> *entries, StableClosure* done)
    check_and_resolve_conflict(entries, done);
```

```cpp
int LogManager::check_and_resolve_conflict(std::vector<LogEntry*> *entries,
                                           StableClosure* done)
    if (entries->front()->id.index == 0) {
        for (size_t i = 0; i < entries->size(); ++i) {
            (*entries)[i]->id.index = ++_last_log_index;
        }
        done_guard.release();
        return 0;
    }

    if (entries->front()->id.index == _last_log_index + 1) {
        _last_log_index = entries->back()->id.index;
    } else {
        // Appending entries overlap the local ones. We should find if there
        // is a conflicting index from which we should truncate the local
        // ones.
        for (; conflicting_index < entries->size(); ++conflicting_index) {

        }


        entries->erase(entries->begin(),
                       entries->begin() + conflicting_index);
    }
```

```cpp
class FollowerStableClosure : public LogManager::StableClosure {
    void run() {

        const int64_t committed_index =
                std::min(_request->committed_index(),
                         // ^^^ committed_index is likely less than the
                         // last_log_index
                         _request->prev_log_index() + _request->entries_size()
                         // ^^^ The logs after the appended entries are
                         // untrustable so we can't commit them even if their
                         // indexes are less than request->committed_index()
                        );

        _node->_ballot_box->set_last_committed_index(committed_index);
    }
}
```

```cpp
int BallotBox::set_last_committed_index(int64_t last_committed_index)
    _last_committed_index.store(last_committed_index, butil::memory_order_relaxed);
    _waiter->on_committed(last_committed_index);
```

```cpp
int FSMCaller::on_committed(int64_t committed_index) {
    ApplyTask t;
    t.type = COMMITTED;
    t.committed_index = committed_index;
    return bthread::execution_queue_execute(_queue_id, t);
}
```

```cpp
int FSMCaller::run(void* meta, bthread::TaskIterator<ApplyTask>& iter)
```