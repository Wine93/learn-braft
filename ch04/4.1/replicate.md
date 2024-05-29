流程详解
===

流程概览
---

1. 前情回顾：当节点成为 Leader 时会为每个 Follower 创建 `Replicator`，并通过发送空的 `AppendEntries` 请求确认各 Follower 的 `nextIndex`
2. 客户端通过 `Node::apply` 接口向 Leader 提交操作日志
3. Leader 向本地追加日志：
   * 3.1 为日志分配 `index`，并将其追加到内存存储中
   * 3.2 异步将内存中的日志持久化到磁盘
4. Leader 将内存中的日志通过 `AppendEntries` 请求并行地复制给所有 Follower
5. Follower 收到 `AppenEntries` 请求，将日志持久化到本地后返回成功响应
6. Leader 若收到大多数确定，则提交日志，更新 `commitIndex` 并应用日志
7. Leader 调用用户状态机的 `on_apply` 应用日志
8. 待 `on_apply` 返回后，更新 `applyIndex`，并删除内存中的日志

<!--
TODO(Wine93)：
流程注解
---

* 整个流程是流水线式的异步实现，非常高效，日志从 `apply` 提交到最后 `on_apply` 被应用，依次经过 `ApplyQueue`、`DiskQueue`、`ApplyTaskQueue` 这 3 个异步队列，详情见以下具体实现
* 前置步骤：`nextIndex` 是下一条要发往 Follower 的日志 `Index`，只有确定了才能往 Follower 发送日志，不然不知道要往 Follower 发送哪些日志
* 1：
* 2.1 日志的 Term 由 Leader 设置为当前的 `Term`；节点刚成为 Leader 时本身拥有的最后一条日志的 `Index` 作为 `LastLogIndex`，往后 Leader 每追加一条日志都将 `++LastLogIndex` 作为该日志的 `Index`
* 2.2 日志的持久化是由管理磁盘的 `bthread` 负责，
* 2.2 和 3 是并行进行的
* 3 日志的发送由单独的 `bthread` 负责，不会阻塞 Leader 处理其他事务
* 4 Follower 端的日志持久化也是异步的
* 5 Leader 的 `CommitIndex` 由 Quorum 机制决定，Follower 的 `CommitIndex` 由 Leader 在下一次的心跳或 `AppendEntries` 请求中携带的 `committed_index` 告知
* 6 通常用户的状态机 `on_apply` 实现需要做 2 件事：(1) 将日志应用到状态机；(2) 将 RPC 响应返回给客户端
* 6 braft 是以串行的方式回调 `on_apply`，所以为了性能，可以
-->

复制模型
---

![图 4.1  复制模型](image/4.1.png)

日志复制是树形复制的模型，采用本地写入和复制同时进行的方式，并且只要大多数写入成功就可以应用日志，不必等待 Leader 本地写入成功。

实现利用异步+并行，全程流水线式处理，并且全链路采用了 Batch，整体十分高效：

* Batch：全链路 Batch，包括持久化、复制以及应用
* 异步：Leader 和 Follower 的日志持久化都是异步的
* 并行：Leader 本地持久化和复制是并行的，并且开启 `pipeline` 后可以保证复制和复制之间也是并行的

关于性能优化的详情，可参考[<4.2 复制优化>](/ch04/4.2/optimization.md)。

Replicator
---

节点在刚成为 Leader 时会为每个 Follower 创建一个 `Replicator`，其运行在单独的 `bthread` 上，主要有以下几个作用：

* 记录 Follower 的一些状态，如 `nextIndex`、`flyingAppendEntriesSize` 等
* 作为 RPC Client，所有从 Leader 发往 Follower 的 RPC 请求都会通过它，包括心跳、`AppendEntriesRequest`、`InstallSnapshotRequest`；
* 同步日志：`Replicator` 会不断地向 Follower 同步日志，直到 Follower 成功复制了 Leader 的所有日志后 ，将在后台等待新日志的到来。

nextIndex
---

`nextIndex` 是 Leader 记录下一个要发往 Follower 的日志 `index`，只有确认了 `nextIndex` 才能给 Follower 发送日志，不然不知道要给 Follower 发送哪些日志。

节点刚成为 Leader 时是不知道每个 Follower 的 `nextIndex` 的，需要通过发送一次或多次探测信息来确认，其探测算法如下：

* (1) `matchIndex` 为最后一条 Leader 与 Follower 匹配的日志，而 `nextIndex=matchIndex+1`
* (2) 初始化 `matchIndex` 为当前 Leader 的最后一条日志的 `index`
* (3) Leader 发送探测请求，携带 `matchIndex` 以及 `matchIndex` 日志对应的 `term`
* (4) Follower 接收请求后，根据自身日志获取 `matchIndex` 对应的 `term`：
  * 若和请求中的 `term` 相等，则代表日志匹配，返回成功响应；
  * 若日志不存在或者 `term` 不匹配，则返回失败响应；
  * 不管成功失败，响应中都携自身的 `lastLogIndex`
* (5) Leader 接收到成功响应，则表示探测成功，确定 `nextIndex`；否则回退 `matchIndex` 并重复步骤 (3)：
    * 若 Follower 的 `lastLogIndex<matchIndex`，则回退 `matchIndex` 为 `lastLogIndex`
    * 否则回退 `matchIndex` 为 `matchIndex-1`

> 下图展示的是一次探测过程，图中的数字代表的是日志的 `term`

![图 4.2  探测过程](image/4.2.png)

日志生命周期
---

* 2.1 日志的 Term 由 Leader 设置为当前的 `Term`；节点刚成为 Leader 时本身拥有的最后一条日志的 `Index` 作为 `LastLogIndex`，往后 Leader 每追加一条日志都将 `++LastLogIndex` 作为该日志的 `Index`
* 2.2 日志的持久化是由管理磁盘的 `bthread` 负责，

commitIndex
---


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
    required int64 prev_log_term = 5;   // matchIndex
    required int64 prev_log_index = 6;  // matchIndex
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

需要注意的是，探测 `nextIndex`、心跳、复制日志都是用的 `append_entries`，区别在于其请求中携带的参数不同：

| 作用     | entries  | committed_index              |
|:---------|:---------|:-----------------------------|
| 探测     | 空       | 0                            |
| 心跳     | 空       | 当前 Leader 的 `CommitIndex` |
| 复制日志 | 携带日志 | 当前 Leader 的 `CommitIndex` |

> 除以上 2 个参数不同外，其余的参数都是一样的

相关接口
---

用户提交任务：
```cpp
class Node {
public:
    // [Thread-safe and wait-free]
    // apply task to the replicated-state-machine
    //
    // About the ownership:
    // |task.data|: for the performance consideration, we will take away the
    //              content. If you want keep the content, copy it before call
    //              this function
    // |task.done|: If the data is successfully committed to the raft group. We
    //              will pass the ownership to StateMachine::on_apply.
    //              Otherwise we will specify the error and call it.
    //
    void apply(const Task& task);
};
```

应用日志：

```cpp
class StateMachine {
public:
    // Update the StateMachine with a batch a tasks that can be accessed
    // through |iterator|.
    //
    // Invoked when one or more tasks that were passed to Node::apply have been
    // committed to the raft group (quorum of the group peers have received
    // those tasks and stored them on the backing storage).
    //
    // Once this function returns to the caller, we will regard all the iterated
    // tasks through |iter| have been successfully applied. And if you didn't
    // apply all the the given tasks, we would regard this as a critical error
    // and report a error whose type is ERROR_TYPE_STATE_MACHINE.
    virtual void on_apply(::braft::Iterator& iter) = 0;
};
```

前置阶段：确定 nextIndex
===

Leader 通过发送空的 `AppendEntries` 来探测 Follower 的 `nextIndex`
只有确定了 `nextIndex` 才能正式向 Follower 发送日志。
这里忽略了很多细节，详情我们将在下一章[复制流程][]中介绍。

Leader 调用 `_send_empty_entries` 向 Follower 发送空的 `AppendEntries` 请求，
并设置响应回调函数为 `_on_rpc_returned`：

```cpp
void Replicator::_send_empty_entries(bool is_heartbeat) {
    ...
    // (1) 填充请求中的字段
    if (_fill_common_fields(
                request.get(), _next_index - 1, is_heartbeat) != 0) {
        ...
    }

    // (2) 设置响应回调函数为 _on_rpc_returned
    google::protobuf::Closure* done = brpc::NewCallback(
                is_heartbeat ? _on_heartbeat_returned : _on_rpc_returned, ...);
    // (3) 发送空的 AppendEntries 请求
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
}

int Replicator::_fill_common_fields(AppendEntriesRequest* request,
                                    int64_t prev_log_index,
                                    bool is_heartbeat) {
    // 获取 Log 的 Term
    const int64_t prev_log_term = _options.log_manager->get_term(prev_log_index);
    ...
    request->set_term(_options.term);
    ...
    request->set_prev_log_index(prev_log_index);  // 当前 leader 的 lastLogIndex
    request->set_prev_log_term(prev_log_term);    // 当前 leader 的 lastLogTerm
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
    ...
    if (request->entries_size() == 0) {  // 响应 send_empty_entries 或心跳
        response->set_success(true);
        response->set_term(_current_term);
        response->set_last_log_index(_log_manager->last_log_index());
        ...
        // see the comments at FollowerStableClosure::run()
        _ballot_box->set_last_committed_index(
                std::min(request->committed_index(),
                         prev_log_index));
        return;
    }
    ...
}
```

阶段一：


阶段一：追加日志
===
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


任务批处理
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
        // (2)
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

阶段二：持久化日志
===

Leader 追加日志
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

持久化日志
---

```cpp
int LogManager::disk_thread(void* meta,
                            bthread::TaskIterator<StableClosure*>& iter) {
}
```

阶段三：复制日志
===

Leader 发送 AE
---

唤醒 Replicator
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

Follower 处理 AE
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

Leader 处理 AE 响应
---

Leader 针对不同的响应

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

阶段四：提交日志
===

提交日志
---

```cpp
int BallotBox::commit_at(
        int64_t first_log_index, int64_t last_log_index, const PeerId& peer) {


    _last_committed_index.store(last_committed_index, butil::memory_order_relaxed);
    _waiter->on_committed(last_committed_index);  // _waiter: FSMCaller

    return 0;
}
```

步骤五：应用日志
===

回调 on_apply
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

清理内存日志
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
===

done 是合适被调用的， 这么判断成功和失败

