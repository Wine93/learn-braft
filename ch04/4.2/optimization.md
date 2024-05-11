优化 0：nextIndex 的探测

优化 1：Batch
===

介绍
---

![batch 优化](image/batch.png)

从客户端调用 `apply` 接口提交 Task 到日志成功复制，并回调用户状态机的 `on_apply` 接口，日志依次经过了 `Apply Queue`、`Disk Queue`、`Apply Task Queue` 这 3 个队列。这些队列都是 brpc 的 [ExecutionQueue][ExecutionQueue] 实现，其消费函数都做了 `batch` 优化：

* **Apply Queue**: 用户提交的 Task 会进入该队列，在该队列的消费函数中会将这些 Task 对应的日志进行打包，并调用 `LogManager` 的 `append_entries` 函数进行追加日志。默认一次最多打包 256 条日志。
* **Disk Queue**: `LogManager` 在接收到这批日志后，需对其进行持久化处理，故会往该队列中提交一个持久化任务（一个任务对应一批日志）；在该队列的消费函数中会将这些任务进行打包，将这批任务对应的所有日志写入磁盘，并在全部写入成功后会做一次 `sync` 操作。默认一次最多打包 256 个持久化任务，而每个任务最多包含 256 条日志，所以其一次 Bacth Write 最多会写入 `256 * 256` 条日志对应的数据。
* **Apply Task Queue**：当日志的复制数（包含持久化）达到 Quorum 后，会调用 `on_committed` 往 `Apply Task Queue` 中提交一个 ApplyTask（每个任务对应一批已提交的日志）；在该队列的消费函数中会将这些 ApplyTask 打包成 `Iterator`，并作为参数回调用户状态机的 `on_apply` 函数。 默认一次最多打包 512 个 ApplyTask，而每个 ApplyTask 最多包含 256 条日志，所以每一次 `on_apply` 参数中的 `Iterator` 最多包含 `512 * 256` 条日志。

[ExecutionQueue]: https://brpc.apache.org/docs/bthread/execution-queue

> **Follower 的 Batch**
>
> 以上讨论的是节点作为 Leader 时的 Batch 优化，当节点为 Follower 时，其优化也是一样的，因为其用的是相同的代码逻辑，唯一的区别在于：
> * **Apply Queue**: Follower 不会接受用户提交的 Task，其日志来源于 Leader 的复制
> * **Disk Queue**: 日志批量落盘的逻辑是一样的，Follower 在接收到 Leader 的一批日志之后也是直接调用 `LogManager` 的 `append_entries` 函数
> * **Apply Task Queue**: 批量 apply 的逻辑是一样的，区别在于调用 `on_committed` 的时机来源于 Leader 在 RPC 中携带的 `committed_index`，并不通过自身的 Quorum 计算

实现
---

优化 2：并行持久化日志
===

介绍
---

![](image/append_parallel.svg)

在 Raft 的实现中，Leader 需要先在本地持久化日志，再向所有的 Follower 复制日志，显然这样的实现具有较高的时延。特别地客户端的写都要经过 Leader，导致 Leader 的压力会变大，从而导致 IO 延迟变高成为慢节点，而本地持久化会阻塞后续的 Follower 复制。所以在 braft 中，Leader 本地持久化和 Follower 复制是并行的，即 Leader 会先将日志写入内存，同时异步地进行持久化和 Follower 复制。

实现
---



优化 3：流水线复制
===

![串行与 Pipeline](image/pipeline-2.svg)


优化 4：raft sync
===

简介
---

braft 在每次 Bacth Write 后都会进行一次 `sync`，显然这会增加时延，所以其提供了一些配置项来控制日志的落盘行为。用户可以根据业务数据丢失的容忍度高低，灵活调整这些配置以达到性能和可靠性之间的权衡。例如对于数据丢失容忍度较高的业务，可以选择将配置项 `raft_sync` 设置为 `flase`这将有助于提升性能。

以下是控制日志 `sync` 的相关参数，前三项针对的是每次的 Batch Write，最后一项针对的是 `Segment`

| 配置项                  | 说明                                                                                                         | 默认值                |
|:------------------------|:-------------------------------------------------------------------------------------------------------------|:----------------------|
| `raft_sync`             | 每次 Batch Write 后是否需要 `sync`                                                                           | `true`                |
| `raft_sync_policy`      | 对于每次 Batch Write 的 sync 策略，有立即 sync （RAFT_SYNC_IMMEDIATELY）和按字节数 sync (RAFT_SYNC_BY_BYTES) | RAFT_SYNC_IMMEDIATELY |
| `raft_sync_per_bytes`   | 在 RAFT_SYNC_BY_BYTES 的同步策略下，满足多少字节需要 sync 一次                                               |
| `raft_sync_segments`    | 每当日志写满一个 `Segment` 需要切换时是否需要 `sync`，每个 Segment 默认存储 8MB 的日志                       | `false`               |
| `raft_max_segment_size` | 单个日志 Segment 大小                                                                                        | 8MB                   |


实现
---

Batch Write 相关配置：

```cpp
int Segment::sync(bool will_sync, bool has_conf) {
    if (_last_index < _first_index) {
        return 0;
    }
    //CHECK(_is_open);
    if (will_sync) {
        if (!FLAGS_raft_sync) {
            return 0;
        }
        if (FLAGS_raft_sync_policy == RaftSyncPolicy::RAFT_SYNC_BY_BYTES
            && FLAGS_raft_sync_per_bytes > _unsynced_bytes
            && !has_conf) {
            return 0;
        }
        _unsynced_bytes = 0;
        return raft_fsync(_fd);
    }
    return 0;
}
```

```cpp
int Segment::close(bool will_sync) {
    ...
    if (_last_index > _first_index) {
        if (FLAGS_raft_sync_segments && will_sync) {
            ret = raft_fsync(_fd);
        }
    }
    ...
}
```

优化 5：异步 Apply
===



参考
===

* [braft docs: 性能优化](https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md#%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
* [braft docs: 复制模型](https://github.com/baidu/braft/blob/master/docs/cn/replication.md)
* [TiKV 功能介绍 – Raft 的优化](https://cn.pingcap.com/blog/optimizing-raft-in-tikv/)
* [Raft 必备的优化手段（二）：Log Replication & others](https://zhuanlan.zhihu.com/p/668511529)
* [持久化策略](https://github.com/baidu/braft/issues?q=is%3Aissue+sync)
