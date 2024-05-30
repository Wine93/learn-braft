优化 1：Batch
===

Batch 队列
---

![图 4.4  Batch 优化](image/4.4.png)

从客户端调用 `apply` 接口提交 `Task` 到日志成功复制，并回调用户状态机的 `on_apply` 接口，日志依次经过了 `Apply Queue`、`Disk Queue`、`Apply Task Queue` 这 3 个队列。这些队列都是 brpc 的 [ExecutionQueue][ExecutionQueue] 实现，其消费函数都做了 `batch` 优化：

* **ApplyQueue**: 用户提交的 `Task` 会进入该队列，在该队列的消费函数中会将这些 `Task` 对应的日志进行打包，并调用 `LogManager` 的 `append_entries` 函数进行追加日志，打包的日志既用于持久化，也用于复制。默认一次打包 32 条日志。
* **DiskQueue**: `LogManager` 在接收到这批日志后，需对其进行持久化处理，故会往该队列中提交一个持久化任务（一个任务对应一批日志）；在该队列的消费函数中会将这些任务进行打包，将这批任务对应的所有日志写入磁盘。默认一次最多打包 256 个持久化任务，而每个任务最多包含 32 条日志，所以其一次 `Bacth Write` 最多会写入 `256 * 32 = 8192` 条日志对应的数据。当然其也受字节数限制，默认每次 `Batch Write` 最多写入 `256KB`。
* **ApplyTaskQueue**：当日志的复制数（包含持久化）达到 `Quorum` 后，会调用 `on_committed` 往 `ApplyTaskQueue` 中提交一个 `ApplyTask`（每个 `ApplyTask` 对应一批已提交的日志）；在该队列的消费函数中会将这些 `ApplyTask` 打包成 `Iterator`，并作为参数回调用户状态机的 `on_apply` 函数。 默认一次最多打包 512 个 `ApplyTask`，而每个 `ApplyTask` 最多包含 32 条日志，所以每一次 `on_apply` 参数中的 `Iterator` 最多包含 `512 * 32 = 16672` 条日志。

从以上看出，日志的复制、持久化、应用，全链路都是经过 `Batch` 优化的。

> **Follower 的 Batch**
>
> 以上讨论的是节点作为 Leader 时的 `Batch` 优化，当节点为 Follower 时，其优化也是一样的，因为其用的是相同的代码逻辑，唯一的区别在于：
> * **ApplyQueue**: Follower 不会接受用户提交的日志（`Task`），其批量日志来源于 Leader 的复制
> * **DiskQueue**: 日志批量落盘的逻辑是一样的，Follower 在接收到 Leader 的一批日志之后也是直接调用 `LogManager` 的 `append_entries` 函数
> * **ApplyTaskQueue**: 批量 `on_apply` 的逻辑是一样的，区别在于 Follower 的 `commitIndex` 来源于 Leader 在 RPC 中携带的 `commitIndex`，并非通过自身的 `Quorum` 计算

相关配置
---

当然，框架也提供了一些配置项来调整这些 `Batch` 大小：

| 作用队列       | 配置项                         | 默认值   | 说明                                                                              |
|:---------------|:-------------------------------|:---------|:----------------------------------------------------------------------------------|
| ApplyQueue     | `raft_apply_batch`             | 32       | 该打包大小影响持久化、落盘以及`on_apply`；上限为 512                              |
| DiskQueue      | `raft_max_append_buffer_size`  | `256 KB` | 每次 `Batch Write` 最大的字节数                                                   |
| ApplyTaskQueue | `raft_fsm_caller_commit_batch` | 512      | 每次 `on_apply` 的最大日志数为 `raft_apply_batch * raft_fsm_caller_commit_batch ` |


[ExecutionQueue]: https://brpc.apache.org/docs/bthread/execution-queue



具体实现
---

`ApplyQueue` 消费函数：
```cpp
int NodeImpl::execute_applying_tasks(
        void* meta, bthread::TaskIterator<LogEntryAndClosure>& iter) {
    ...
    // (1) FLAGS_raft_apply_batch 默认为 32
    const size_t batch_size = FLAGS_raft_apply_batch;
    // (2) 定义一个 tasks 数组，数组大小 = min(batch_size, 256)
    DEFINE_SMALL_ARRAY(LogEntryAndClosure, tasks, batch_size, 256);
    size_t cur_size = 0;
    NodeImpl* m = (NodeImpl*)meta;
    for (; iter; ++iter) {
        if (cur_size == batch_size) {
            m->apply(tasks, cur_size);  // (4) 调用批量 apply 函数
            cur_size = 0;
        }
        tasks[cur_size++] = *iter;  // (3) batch 打包
    }
    ...
    return 0;
}
```

`DiskQueue` 消费函数：

```cpp
int LogManager::disk_thread(void* meta,
                            bthread::TaskIterator<StableClosure*>& iter) {
    ...
    // (1) 定义 AppendBatcher 的 Batch 大小为 256
    StableClosure* storage[256];
    AppendBatcher ab(storage, ARRAY_SIZE(storage), &last_id, log_manager);

    for (; iter; ++iter) {
                // ^^^ Must iterate to the end to release to corresponding
                //     even if some error has occurred
        StableClosure* done = *iter;
        ...
        if (!done->_entries.empty()) {
            ab.append(done);  // (2) batch 打包持久化任务
        } else {
            ab.flush();  // (3) 批量持久化
            ...
        }
    }
    ...
    return 0;
}

class AppendBatcher {
public:
    ...
    void flush() {
        if (_size > 0) {
            // 写入持久化存储
            _lm->append_to_storage(&_to_append, _last_id, &metric);
            ...
        }
        ...
    }

    void append(LogManager::StableClosure* done) {
        // FLAGS_raft_max_append_buffer_size 默认为 256KB
        // 一次 Batch Write 写入的字节数不能超过该配置值
        if (_size == _cap ||
                _buffer_size >= (size_t)FLAGS_raft_max_append_buffer_size) {
            flush();
        }
        _storage[_size++] = done;
        _to_append.insert(_to_append.end(),
                         done->_entries.begin(), done->_entries.end());
        for (size_t i = 0; i < done->_entries.size(); ++i) {
            _buffer_size += done->_entries[i]->data.length();
        }
    }
}
```

`ApplyTaskQueue` 消费函数：


```cpp
int FSMCaller::run(void* meta, bthread::TaskIterator<ApplyTask>& iter) {
    FSMCaller* caller = (FSMCaller*)meta;
    ...
    int64_t max_committed_index = -1;
    int64_t counter = 0;
    // FLAGS_raft_fsm_caller_commit_batch 默认为 512
    size_t  batch_size = FLAGS_raft_fsm_caller_commit_batch;
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
                batch_size = FLAGS_raft_fsm_caller_commit_batch;
            }
            ...
        }
    }
    ...
    return 0;
}

// 生产 Iterator，调用用户状态机的 `on_apply`
void FSMCaller::do_committed(int64_t committed_index) {
    ...
    IteratorImpl iter_impl(_fsm, _log_manager, &closure, first_closure_index,
                 last_applied_index, committed_index, &_applying_index);
    for (; iter_impl.is_good();) {
        ...
        Iterator iter(&iter_impl);
        _fsm->on_apply(iter);
        ...
        // Try move to next in case that we pass the same log twice.
        iter.next();
    }
    ...
}
```

优化 2：并行持久化日志
===

持久化与复制
---

![图 4.5  持久化日志的串行与并行](image/4.5.png)

在 Raft 的实现中，Leader 需要先在本地持久化日志，再向所有的 Follower 复制日志，显然这样的实现具有较高的时延。特别地客户端的写都要经过 Leader，导致 Leader 的压力会变大，从而导致 `IO` 延迟变高成为慢节点，而本地持久化也会阻塞后续的 Follower 复制。

所以在 braft 中，Leader 本地持久化和 Follower 复制是并行的，即 Leader 会先将日志写入内存，同时异步地进行持久化和向 Follower 复制。并且只要大多数节点写入成功就可以应用日志，不必等待 Leader 本地持久化成功。

具体实现
---

具体的实现我们已经在上一节中详细介绍过了，参见 [<4.1 复制流程>](/ch04/4.1/replicate.md)。

优化 3：流水线复制
===

pipeline
---

![图 4.6  串行与 Pipeline](image/4.6.png)

Raft 默认是串行复制日志，显然这样效率是比较低下的。可以采用 `Pipeline` 的复制方式，即 Leader 发送完 `AppendEntries` 请求后不必等待其响应，立马发送一下批日志。当然，这样的实现对于接受端（Follower）来说，可能会带来乱序、空洞等问题，为此，braft 在 Follower 端引入了日志缓存，将不是顺序的日志先缓存起来，待其前面的日志都接受到后再写入该日志，以达到顺序写入磁盘的目的。

特别需要注意的是，`pipeline` 优化默认是关闭的，需要用户通过以下 2 个配置项开启：
```cpp
DEFINE_bool(raft_enable_append_entries_cache, false,
            "enable cache for out-of-order append entries requests, should used when "
            "pipeline replication is enabled (raft_max_parallel_append_entries_rpc_num > 1).");

DEFINE_int32(raft_max_parallel_append_entries_rpc_num, 1,
             "The max number of parallel AppendEntries requests");
```

具体实现
---

Leader 发送 `AppendEntries`：

```cpp
void Replicator::_send_entries() {
    ...
    // 发送 AppendEntries
    google::protobuf::Closure* done = brpc::NewCallback(
                _on_rpc_returned, _id.value, cntl.get(),
                request.get(), response.get(), butil::monotonic_time_ms());
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
    // 注册 waiter 等待新日志的到来，如果目前还有日志未发送，会立马唤醒 waiter 继续发送日志
    _wait_more_entries();
}

// waiter 被唤醒会调用 _continue_sending 继续发送日志
int Replicator::_continue_sending(void* arg, int error_code) {
    ...
    r->_send_entries();
    ...
}
```

Follower 处理 `AppendEntries` 请求：

* 对于到来的日志，如果其前面的日志没有接受到，先将其缓存起来，并要求客户端重发前面的日志
* 待前面的日志都达到后，再将缓存中在其之后的日志写入磁盘

```cpp
void NodeImpl::handle_append_entries_request(brpc::Controller* cntl,
                                             const AppendEntriesRequest* request,
                                             AppendEntriesResponse* response,
                                             google::protobuf::Closure* done,
                                             bool from_append_entries_cache) {
    // 当到来的日志的前面的日志没有接受到时，将其缓存起来
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
            ...
            return;
        }

        // 设为 false, 要求 Leader 重发前面的日志
        response->set_success(false);
        response->set_term(_current_term);
        response->set_last_log_index(last_index);  // 响应当前 follower 的最后一条日志的 index
        ...
        return;
    }

    // (2) 当前的日志前面的日志都已达到，可以追加，这时候查看缓存中是否还有之后的日志，如果是顺序的，将以缓存的内容调用 handle_append_entries_request，将这些日志一并写入磁盘
    // check out-of-order cache
    check_append_entries_cache(index);

    FollowerStableClosure* c = new FollowerStableClosure(
            cntl, request, response, done_guard.release(),
            this, _current_term);
    _log_manager->append_entries(&entries, c);
}
```

优化 4：raft sync
===

sync 配置
---

日志每次 `Batch Write` 是先将日志写入 `Page Cache`，最后再调用一次 `fsync` 操作。显然 `sync` 操作会增加时延，为此 braft 提供了一些配置项来控制日志的 `sync` 行为。用户可以根据业务数据丢失的容忍度高低，灵活调整这些配置以达到性能和可靠性之间的权衡。例如对于数据丢失容忍度较高的业务，可以选择将配置项 `raft_sync` 设置为 `Flase`，这将有助于提升性能。

以下是控制日志 `sync` 的相关参数，前三项针对的是每次的 `Batch Write`，最后两项针对的是 `Segment`：

| 配置项                  | 说明                                                                                                                       | 默认值                  |
|:------------------------|:---------------------------------------------------------------------------------------------------------------------------|:------------------------|
| `raft_sync`             | 每次 `Batch Write` 后是否需要 `sync`                                                                                       | `True`                  |
| `raft_sync_policy`      | 对于每次 `Batch Write` 后的 `sync` 策略，有立即 `sync` （`RAFT_SYNC_IMMEDIATELY`）和按字节数 `sync` (`RAFT_SYNC_BY_BYTES`) | `RAFT_SYNC_IMMEDIATELY` |
| `raft_sync_per_bytes`   | 在 `RAFT_SYNC_BY_BYTES` 的策略下，每多少字节 `sync` 一次                                                                   | `INT32_MAX`             |
| `raft_sync_segments`    | 每当日志写满一个 `Segment` 需要切换时是否需要 `sync`，每个 `Segment` 默认存储 `8MB` 的日志                                 | `False`                 |
| `raft_max_segment_size` | 单个日志 `Segment` 大小                                                                                                    | `8MB`                   |


具体实现
---

`Batch Write` 写入 `Page Cache` 后会调用 `Segment::sync` 进行 `sync` 操作：

```cpp
int Segment::sync(bool will_sync, bool has_conf) {
    ...
    if (will_sync) {
        // (1) 如果 `raft_sync` 为 False，则直接返回
        if (!FLAGS_raft_sync) {
            return 0;
        }

        // (2) 如果是按字节 `sync` 并且当前未 `sync` 的字节小于配置值，则直接返回
        if (FLAGS_raft_sync_policy == RaftSyncPolicy::RAFT_SYNC_BY_BYTES
            && FLAGS_raft_sync_per_bytes > _unsynced_bytes
            && !has_conf) {
            return 0;
        }
        _unsynced_bytes = 0;

        // (3) 调用 fsync(fd)
        return raft_fsync(_fd);
    }
    return 0;
}
```

每当一个 `Segment` 文件写完后，都会调用 `Segment::close` 将其转换成 `closed segment`：

```cpp
int Segment::close(bool will_sync) {
    ...
    if (FLAGS_raft_sync_segments && will_sync) {
        ret = raft_fsync(_fd);
    }
    ...
}
```

优化 5：异步 Apply
===

```cpp
void on_apply(braft::Iterator& iter) {
    // A batch of tasks are committed, which must be processed through
    // |iter|
    for (; iter.valid(); iter.next()) {
        // This guard helps invoke iter.done()->Run() asynchronously to
        // avoid that callback blocks the StateMachine.
        braft::AsyncClosureGuard closure_guard(iter.done());
        // Parse operation from iter.data() and execute this operation
        // op = parse(iter.data());
        // result = process(op)

        // The purpose of following logs is to help you understand the way
        // this StateMachine works.
        // Remove these logs in performance-sensitive servers.
        LOG_IF(INFO, FLAGS_log_applied_task)
                << "Exeucted operation " << op
                << " and the result is " << result
                << " at log_index=" << iter.index();
    }
}
```

当日志被提交时，框架会串行调用用户状态机的 `on_apply`，虽然这里做了 `Batch` 优化，但是对于那些不支持批量更新的状态机来说，仍然是低效的。为此，用户可以将 `on_apply` 函数异步执行，让那些可以并行的操作尽可能并行起来。

需要注意的是，当 `on_apply` 异步后，需要处理好节点成为 Leader 时日志回放的问题。当节点刚成为 Leader 时，需要回放（`on_apply`）之前任期的日志，这时候需要将这些日志全部应用到状态机后才能处理读取操作，不然可能会违背线性一致性。因为这些日志在之前的任期可能被提交了，客户端能读取到，而在新任期回放的时候，由于这些日志是异步执行的（`on_apply` 返回了，但还在异步执行中），可能还没应用到状态机，这时候客户端去读取可能是读取不到的。

参考
===

* [braft docs: 性能优化](https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md#%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
* [braft docs: 复制模型](https://github.com/baidu/braft/blob/master/docs/cn/replication.md)
* [TiKV 功能介绍 – Raft 的优化](https://cn.pingcap.com/blog/optimizing-raft-in-tikv/)
* [Raft 必备的优化手段（二）：Log Replication & others](https://zhuanlan.zhihu.com/p/668511529)
* [持久化策略](https://github.com/baidu/braft/issues?q=is%3Aissue+sync)
