流程详解
===

流程概览
---

0. 前置：当节点成为 Leader 时会通过发送空的 `AppendEntries` 请求确认各 Follower 的 `nextIndex`
1. 客户端通过 `apply` 接口向 Leader 提交操作日志
2. Leader 向本地追加日志：
   * 2.1 为日志分配 `Index`，并将其追加到内存存储中
   * 2.2 异步将内存中的日志持久化到磁盘
3. Leader 将内存中的日志通过 `AppendEntries` 请求并行地发送给所有 Follower
4. Follower 收到 `AppenEntries` 请求，将日志持久化到本地后返回成功响应
5. Leader 若收到大多数确定，则提交日志，更新 `CommitIndex`
5. Leader 回调用户状态机的 `on_apply` 应用日志
7. 待 `on_apply` 返回后，更新 `ApplyIndex`，并删除内存中的日志

流程注解
---

* 整个流程是流水线式的异步实现，非常高效，日志从 `apply` 提交到最后 `on_apply` 被应用，依次经过 `ApplyQueue`、`DiskQueue`、`ApplyTaskQueue` 这 3 个异步队列，详情见以下具体实现
* 0：`nextIndex` 是下一条要发往 Follower 的日志 `Index`，只有确定了才能往 Follower 发送日志，不然不知道要往 Follower 发送哪些日志
* 1：
* 2.1 日志的 Term 在用户提交时指定；节点成为 Leader 时本身拥有的最后一条日志的 `Index` 作为 `LastLogIndex`，往后 Leader 每追加一条日志都将 `++LastLogIndex` 作为该日志的 `Index`
* 2.2 日志的持久化是由管理磁盘的 `bthread` 负责，
* 2.2 和 3 是并行进行的
* 3 日志的发送由单独的 `bthread` 负责，不会阻塞 Leader 处理其他事务
* 4 Follower 端的日志持久化也是异步的
* 5 Leader 的 `CommitIndex` 由 Quorum 机制决定，Follower 的 `CommitIndex` 由 Leader 在下一次的心跳或 `AppendEntries` 请求中携带的 `committed_index` 告知
* 6 通常用户的状态机 `on_apply` 实现需要做 2 件事：(1) 将日志应用到状态机；(2) 将 RPC 响应返回给客户端
* 6 braft 是以串行的方式回调 `on_apply`，所以为了性能，可以

相关 RPC
---

```proto
enum EntryType {
    ENTRY_TYPE_UNKNOWN = 0;
    ENTRY_TYPE_NO_OP = 1;
    ENTRY_TYPE_DATA = 2;
    ENTRY_TYPE_CONFIGURATION= 3;
};

message EntryMeta {
    required int64 term = 1;
    required EntryType type = 2;
    repeated string peers = 3;
    optional int64 data_len = 4;
    // Don't change field id of `old_peers' in the consideration of backward
    // compatibility
    repeated string old_peers = 5;
};

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

service RaftService {
    rpc append_entries(AppendEntriesRequest) returns (AppendEntriesResponse);
};
```

| 作用           | entries | committed_index |  |  |  |
|:---------------|:--------|:----------------|:-|:-|:-|
| 探测 `nextIndex` |
| 心跳           |
| 复制日志       |


整体流程
===

Leader 侧流程
---
![日志复制整体流程](image/replication-2.svg)

Follower 侧流程
---

*Follower* 接受到 *AppendEntries* 请求后，将直接调用 *LogManager* 的 `AppendEntries` 接口追加日志，除了几下几点差异，其处理流程和上图中的 *Leader* 侧流程基本一致：

*
* *Leader* 在往内存中插入日志会，后唤醒所有的 `Replicator` 给其对应的 *Follower* 发送 *AppendEntries* 请求，而 *Follower* 没有任何 *Replicator*

日志复制失败
---

```cpp
void function(op, callback) {
    braft::Task task;
    ...
    task.done = make_closure(callback);
    ...
    return _node->apply(task);
}
```

用户在构建 *task* 的时候需要指定对应的 *callback*
* 如果日志复制成功，需要用户在 `on_apply` 接口中回调 *callback*
* 如果日志复制失败，将由 *braft* 回调 *callback*，并将 *error* 信息传递给 *callback*

*braft* 在以下

客户端在调用 *apply* 的时候，如果该操作对应的日志被成功被复制并回调 `on_apply`，需要用户在 `on_apply` 接口中主动

当当前 *Leader* 已不是 *Leader* 时，主要发生在以下几个场景：

* 响应: term > current_term

```cpp
status.set_error(EHIGHERTERMRESPONSE, "Leader receives higher term "
                    "%s from peer:%s", response->GetTypeName().c_str(), r->_options.peer_id.to_string().c_str());

```

具体实现
===

步骤一：用户提交任务
---
```cpp
#include <braft/raft.h>

...
void function(op, callback) {
    butil::IOBuf data;
    serialize(op, &data);
    braft::Task task;
    // The data applied to StateMachine
    task.data = &data;
    // Continuation when the data is applied to StateMachine or error occurs.
    task.done = make_closure(callback);
    // Reject this task if expected_term doesn't match the current term of
    // this Node if the value is not -1
    task.expected_term = expected_term;
    return _node->apply(task);
}
```

客户端需要将操作序列化成 [IOBuf][IOBuf]，并构建一个 *Task* 向 *braft::Node* 提交。

[IOBuf]: https://github.com/apache/brpc/blob/master/src/butil/iobuf.h

步骤二: 任务批处理
---

该阶段

* 节点收到 *task* 后，会将其转换成 `LogEntryAndClosure` 并放入 `_apply_queue` 中。至此，客户端的 *apply* 就完成返回了
* 在队列的消费函数 `execute_applying_tasks` 中，会将这些 *task* 打包成 *tasks* 并交给 *bacth apply* 接口处理。默认 *tasks* 包含 256 个 *task*
* *bacth apply* 接收到 tasks 后会执行以下这些动作：
    * 将 *task* 转换成 *LogEntry* 并填充 *term* 和 *type*
    * **初始化 Ballot**：调用 `BallotBox::append_pending_task` 为每一个 *LogEntry* 添加一个 `Ballot`，该 `Ballot` 主要用于计数，当 *LogEntry* 成功被持久化或每被一个 *Follower* Append 后，都会调用 `BallotBox::commit_at` 将计数加一，当计数达到 `quorum` 后，则会回调 `on_apply`
    * **追加日志**：调用 `LogManager::append_entries` 接口进行追加日志，在该接口中会对日志进行持久化存储，并唤醒 `Replicator` 将日志发送给 *Follower*

*apply* 接口：

```cpp
void NodeImpl::apply(const Task& task) {
    ...
    LogEntryAndClosure m;
    m.entry = entry;  // m.entry = task.data
    m.done = task.done;
    m.expected_term = task.expected_term;
    if (_apply_queue->execute(m, &bthread::TASK_OPTIONS_INPLACE, NULL) != 0) {
        ...
    }
}
```

队列消费函数:

```cpp
int NodeImpl::execute_applying_tasks(void* meta,  bthread::TaskIterator<LogEntryAndClosure>& iter) {
    NodeImpl* m = (NodeImpl*)meta;
    for (; iter; ++iter) {
        if (cur_size == batch_size) {  // batch_size = 256
            m->apply(tasks, cur_size);
            cur_size = 0;
        }
        tasks[cur_size++] = *iter;
    }
    ...
}
```

批量 *apply* 接口:

```cpp
void NodeImpl::apply(LogEntryAndClosure tasks[], size_t size) {
    std::vector<LogEntry*> entries;
    ...
    for (size_t i = 0; i < size; ++i) {
        ...
        entries.push_back(tasks[i].entry);
        entries.back()->id.term = _current_term;
        entries.back()->type = ENTRY_TYPE_DATA;
        ...
        _ballot_box->append_pending_task(_conf.conf,
                                         _conf.stable() ? NULL : &_conf.old_conf,
                                         tasks[i].done);
    }
    ...
    _log_manager->append_entries(&entries,
                               new LeaderStableClosure(
                                        NodeId(_group_id, _server_id),
                                        entries.size(),
                                        _ballot_box));
    ...
}
```

步骤三: Leader 追加日志
---

*LogManager* 是 *braft* 管理日志的入口，

* 详见[<步骤四: Leader 持久化日志>]()
* 详见[<步骤五: Leader 发送 AE>]()

所以步骤四、五是并发执行的。

```cpp
void LogManager::append_entries(std::vector<LogEntry*> *entries, StableClosure* done) {
    ...
    // check_and_resolve_conflict 会给每一个 LogEntry 分配 index
    if (!entries->empty() && check_and_resolve_conflict(entries, done) != 0) {
        ...
        return;
    }

    if (!entries->empty()) {
        _logs_in_memory.insert(_logs_in_memory.end(), entries->begin(), entries->end());
    }

    ...

    // done: LeaderStableClosure
    int ret = bthread::execution_queue_execute(_disk_queue, done);
    wakeup_all_waiter(lck);
}
```

唤醒所有 *waiter*：

```cpp
void LogManager::wakeup_all_waiter(std::unique_lock<raft_mutex_t>& lck) {
    ...
    for (size_t i = 0; i < nwm; ++i) {
        ...
        if (bthread_start_background( &tid, &attr, run_on_new_log, wm[i]) != 0) { ...
        }
    }
}

void* LogManager::run_on_new_log(void *arg) {
    ...
    // wm: Replicator
    // on_new_log: _continue_sending
    wm->on_new_log(wm->arg, wm->error_code);
    ...
}
```

步骤四: Leader 持久化日志
---

```cpp
int LogManager::disk_thread(void* meta,
                            bthread::TaskIterator<StableClosure*>& iter) {
}
```

步骤五: Leader 发送 AE
---

*Replicator* 被唤醒后会调用 `_continue_sending` 继续发送 *AppendEntries* 请求。在 `_send_entries` 函数中主要做以下几件事情：

* 调用 `_fill_common_fields` 填充 *request*
```proto
message AppendEntriesRequest {
    required string group_id = 1;   // Raft Group Id
    required string server_id = 2;  // 发送成员 PeerId（即 Leader PeerId)
    required string peer_id = 3;    // 接受成员 PeerId
    required int64 term = 4;        // Leader term
    required int64 prev_log_term = 5;
    required int64 prev_log_index = 6;
    repeated EntryMeta entries = 7;
    required int64 committed_index = 8;
};
```
* 调用 `_wait_more_entries`


*Replicator* 被唤醒的回调函数：
```cpp
int Replicator::_continue_sending(void* arg, int error_code) {
    ...
    r->_send_entries();
    ...
}
```

发送 *AppendEntries* 请求：

```cpp
void Replicator::_send_entries() {
    if (_fill_common_fields(request.get(), _next_index - 1, false) != 0) {
        ...
        return _install_snapshot();
    }
    ...
    _next_index += request->entries_size();
    ...
    google::protobuf::Closure* done = brpc::NewCallback( _on_rpc_returned, ...);
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
    _wait_more_entries();
}
```

```cpp
int Replicator::_fill_common_fields(AppendEntriesRequest* request,
                                    int64_t prev_log_index,  // leader 中记录该 follower 的 next_log_id
                                    bool is_heartbeat) {
    const int64_t prev_log_term = _options.log_manager->get_term(prev_log_index);
    // 查不到对应日志的 term，代表已经被快照压缩了
    if (prev_log_term == 0 && prev_log_index != 0) {
        ...
        return -1;
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

步骤六: Follower 处理 AE
---

* 当日志被成功持久化后，会调用 `FollowerStableClosure`

```cpp
void NodeImpl::handle_append_entries_request(brpc::Controller* cntl,
                                             const AppendEntriesRequest* request,
                                             AppendEntriesResponse* response,
                                             google::protobuf::Closure* done,
                                             bool from_append_entries_cache) {
    if (request->term() < _current_term) {
        const int64_t saved_current_term = _current_term;
        ...
        response->set_success(false);
        response->set_term(saved_current_term);
        return;
    }

    FollowerStableClosure* c = new FollowerStableClosure(
            cntl, request, response, done_guard.release(),
            this, _current_term);
    _log_manager->append_entries(&entries, c);
}
```

```cpp
class FollowerStableClosure : public LogManager::StableClosure {
public:
    ...
    void Run() {
        run();
        delete this;
    }
private:
    ...
    void run() {
        brpc::ClosureGuard done_guard(_done);
        if (!status().ok()) {
            _cntl->SetFailed(status().error_code(), "%s",
                             status().error_cstr());
            return;
        }
        std::unique_lock<raft_mutex_t> lck(_node->_mutex);
        if (_term != _node->_current_term) {
            // The change of term indicates that leader has been changed during
            // appending entries, so we can't respond ok to the old leader
            // because we are not sure if the appended logs would be truncated
            // by the new leader:
            //  - If they won't be truncated and we respond failure to the old
            //    leader, the new leader would know that they are stored in this
            //    peer and they will be eventually committed when the new leader
            //    found that quorum of the cluster have stored.
            //  - If they will be truncated and we responded success to the old
            //    leader, the old leader would possibly regard those entries as
            //    committed (very likely in a 3-nodes cluster) and respond
            //    success to the clients, which would break the rule that
            //    committed entries would never be truncated.
            // So we have to respond failure to the old leader and set the new
            // term to make it stepped down if it didn't.
            _response->set_success(false);
            _response->set_term(_node->_current_term);
            return;
        }
        // It's safe to release lck as we know everything is ok at this point.
        lck.unlock();

        // DON'T touch _node any more
        _response->set_success(true);
        _response->set_term(_term);

        const int64_t committed_index =
                std::min(_request->committed_index(),
                         // ^^^ committed_index is likely less than the
                         // last_log_index
                         _request->prev_log_index() + _request->entries_size()
                         // ^^^ The logs after the appended entries are
                         // untrustable so we can't commit them even if their
                         // indexes are less than request->committed_index()
                        );
        //_ballot_box is thread safe and tolerates disorder.
        _node->_ballot_box->set_last_committed_index(committed_index);  // 这里会调用用户的 |on_apply|
    }
};
```

```cpp
int BallotBox::set_last_committed_index(int64_t last_committed_index) {
    ...
    if (last_committed_index > _last_committed_index.load(...)) {
        _last_committed_index.store(last_committed_index, ...);
        _waiter->on_committed(last_committed_index);  // _waiter: FSMCaller
    }
    return 0;
}
```

步骤七: Leader 处理 AE 响应
---

*Leader* 针对不同的响应

* **RPC 失败**：调用 `_block` 阻塞当前 *Replicator* 一段时间（默认 100 毫秒），超时后调用 `_continue_sending` 重新发送当前 *AppendEntries* 请求。出现这种情况一般是对应的 *Follower* crash 了，需要不断重试直到其恢复正常或被剔除集群。
* **响应失败**：这里又细分为 2 种情况
    * term: `step_down` 降为 follower
    * index: 回退 `_next_index`
* **响应成功**：

```cpp
void Replicator::_on_rpc_returned(ReplicatorId id, brpc::Controller* cntl,
                                  AppendEntriesRequest* request,
                                  AppendEntriesResponse* response,
                                  int64_t rpc_send_time) {
    // 情况 1：RPC 请求失败
    if (cntl->Failed()) {
        ...
        r->_reset_next_index();
        return r->_block(start_time_us, cntl->ErrorCode());
    }

    ...

    // 情况 3：响应成功
    if (entries_size > 0) {
        r->_options.ballot_box->commit_at(
                min_flying_index, rpc_last_log_index,
                r->_options.peer_id);
        ...
    }
}
```

步骤八：投票箱计数
---

```cpp
int BallotBox::commit_at(
        int64_t first_log_index, int64_t last_log_index, const PeerId& peer) {


    _last_committed_index.store(last_committed_index, butil::memory_order_relaxed);
    _waiter->on_committed(last_committed_index);  // _waiter: FSMCaller

    return 0;
}
```

步骤九: 回调 `on_apply`
---

```cpp
int FSMCaller::on_committed(int64_t committed_index) {
    ApplyTask t;
    t.type = COMMITTED;
    t.committed_index = committed_index;
    return bthread::execution_queue_execute(_queue_id, t);
}
```

```cpp
int FSMCaller::run(void* meta, bthread::TaskIterator<ApplyTask>& iter) {
    for (; iter; ++iter) {
        if (iter->type == COMMITTED && counter < batch_size) {
            if (iter->committed_index > max_committed_index) {
                max_committed_index = iter->committed_index;
                counter++;
            }
        } else {
            if (max_committed_index >= 0) {
                caller->_cur_task = COMMITTED;
                caller->do_committed(max_committed_index);
                max_committed_index = -1;
                counter = 0;
            }
            ...
        }
        ...
    }
}
```

```cpp
void FSMCaller::do_committed(int64_t committed_index) {
    IteratorImpl iter_impl(_fsm, _log_manager, ...);
    for (; iter_impl.is_good();) {
        ...
        Iterator iter(&iter_impl);
        _fsm->on_apply(iter);  // _fsm: StateMachine
        ...
        iter.next();
    }

    ...

    LogId last_applied_id(last_index, last_term);
    _last_applied_index.store(committed_index, butil::memory_order_release);
    _last_applied_term = last_term;
    _log_manager->set_applied_id(last_applied_id);
}
```

步骤十: 清理内存日志
---

```cpp
void LogManager::set_applied_id(const LogId& applied_id) {
    std::unique_lock<raft_mutex_t> lck(_mutex);  // Race with set_disk_id
    if (applied_id < _applied_id) {
        return;
    }
    _applied_id = applied_id;
    LogId clear_id = std::min(_disk_id, _applied_id);
    lck.unlock();
    return clear_memory_logs(clear_id);
}
```

其他：日志复制失败
---

* 清理内存中的日志

```cpp
LogManager::set_applied_id()
```


done 是合适被调用的， 这么判断成功和失败

