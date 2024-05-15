
流程详解
===

流程概览
---

快照有压缩日志和加速启动的作用，其创建流程如下：

1. 当快超 `Timer` 超时或用户手动触发，会触发节点执行创建快照的任务
2. 节点会创建一 `temp` 目录来保存临时快照，并返回一个 `SnapshotWriter`
3. 回调用户状态机的 `on_snapshot_save`
    * 3.1 `SnapshotWriter`
4. 用户需通过 `SnapshotWriter` 将状态数据写入到临时快照中
5. 待快照写入完成，用户需回调 `Closure`
    * 5.1 写入快照元数据
    * 5.2 通过 `rename()` 将临时快照转换为正式快照
    * 5.3 删除上一个快照
    * 5.4 删除上一个快照对应的日志
6. 至此，快照创建完成

流程整体分为以下 3 个阶段：创建临时快照（1-2），用户写入数据（3-4），转换为正式快照（5-6）

![图 5.1  创建快照整体流程](image/snapshot_save.svg)

流程注解
---

* 1：除了节点的定时快照，用户可手动触发快照；Leader 和 Follower 各自执行快照任务
* 2.1
* 5.2 正式的快照目录名以当时创建快照时的 `ApplyIndex` 命名，如 `snapshot_00000000000000001000`
* 5.4：

异步快照
---

```cpp

void on_snapshot_save() {
}
```

快照元数据
---

```proto
message SnapshotMeta {
    required int64 last_included_index = 1;
    required int64 last_included_term = 2;
    repeated string peers = 3;
    repeated string old_peers = 4;
}

message LocalFileMeta {
    optional bytes user_meta   = 1;
    optional FileSource source = 2;
    optional string checksum   = 3;
}

message LocalSnapshotPbMeta {
    message File {
        required string name = 1;
        optional LocalFileMeta meta = 2;
    };
    optional SnapshotMeta meta = 1;
    repeated File files = 2;
}
```

相关接口
---

```cpp
class Node {
public:
    // Start a snapshot immediately if possible. done->Run() would be invoked
    // when the snapshot finishes, describing the detailed result.
    void snapshot(Closure* done);
};
```

```cpp
class StateMachine {
public:
    // user defined snapshot generate function, this method will block on_apply.
    // user can make snapshot async when fsm can be cow(copy-on-write).
    // call done->Run() when snapshot finished.
    // success return 0, fail return errno
    // Default: Save nothing and returns error.
    virtual void on_snapshot_save(::braft::SnapshotWriter* writer,
                                  ::braft::Closure* done);
};
```

```cpp
class SnapshotWriter : public Snapshot {
public:
    // Add a file to the snapshot.
    // |file_meta| is an implmentation-defined protobuf message
    // All the implementation must handle the case that |file_meta| is NULL and
    // no error can be raised.
    // Note that whether the file will be created onto the backing storage is
    // implementation-defined.
    virtual int add_file(const std::string& filename);

    // Remove a file from the snapshot
    // Note that whether the file will be removed from the backing storage is
    // implementation-defined.
    virtual int remove_file(const std::string& filename) = 0;
};
```

阶段一：创建临时快照
===

触发快照任务
---

节点在初始化时就已经启动了快照的 *timer*，当 *timer timeout* 时就会触发快照任务。节点会往任务队列 *ApplyQueue* 中加入一个创建快照（*SNAPSHOT_SAVE*）的任务，待队列，其主要执行以下动作：


```cpp
void SnapshotTimer::run() {
    _node->handle_snapshot_timeout();
}
```

```cpp
void NodeImpl::handle_snapshot_timeout() {
}
```

```cpp
void NodeImpl::do_snapshot(Closure* done) {
    LOG(INFO) << "node " << _group_id << ":" << _server_id
              << " starts to do snapshot";
    if (_snapshot_executor) {
        _snapshot_executor->do_snapshot(done);
    } else {
        if (done) {
            done->status().set_error(EINVAL, "Snapshot is not supported");
            run_closure_in_bthread(done);
        }
    }
}
```

```cpp
void SnapshotExecutor::do_snapshot(Closure* done) {
    // check snapshot install/load
    if (_downloading_snapshot.load(butil::memory_order_relaxed)) {
        ...
        return;
    }

    // check snapshot saving?
    if (_saving_snapshot) {
        ...
        return;
    }
    _saving_snapshot = true;

    SaveSnapshotDone* snapshot_save_done = new SaveSnapshotDone(this, writer, done);
    if (_fsm_caller->on_snapshot_save(snapshot_save_done) != 0) {
    }
}
```


阶段二：用户写入数据
===

用户需要在

用户需要确保在调用 `done->Run` 前，用户需要调用 `sync` 确保数据已经落盘，否则可能会导致数据丢失。

阶段三：转为正式快照
===

写入元数据
---

删除上一个快照
---

`on_snapshot_save_done` 主要做以下几件事情：

* 保存 *snapshot meta* 到 `__raft_snapshot_meta` 文件中，其通过将 *proto* 文件序列化后，包括快照中所有文件的列表以及每个每个文件的 *checksum*，需要注意的是 last_included_* 字段是快照启动时的 *index* 和 *term*
```proto
message SnapshotMeta {
    required int64 last_included_index = 1;  // 快照对应最后一条日志的 index
    required int64 last_included_term = 2;  // 快照对应最后一条日志的 index
    repeated string peers = 3;  // 当前集群的 peer 列表
    repeated string old_peers = 4;  // 默认为空。配置更变时，即 C{old,new}
}

enum FileSource {
    FILE_SOURCE_LOCAL = 0;
    FILE_SOURCE_REFERENCE = 1;
}

message LocalFileMeta {
    optional bytes user_meta   = 1;
    optional FileSource source = 2;
    optional string checksum   = 3;
}

message SnapshotMeta {
    required int64 last_included_index = 1;
    required int64 last_included_term = 2;
    repeated string peers = 3;
    repeated string old_peers = 4;
}

message LocalSnapshotPbMeta {
    message File {
        required string name = 1;
        optional LocalFileMeta meta = 2;
    };
    optional SnapshotMeta meta = 1;
    repeated File files = 2;
}
```

```cpp
int SnapshotExecutor::on_snapshot_save_done(
    const butil::Status& st, const SnapshotMeta& meta, SnapshotWriter* writer) {

}
```



* rename
* 删除上一个快照，特别需要注意的时，每一个快照都有一个 referer，代表还有 follower 在下载这个快照
* 删除倒数第二个快照（即当前快照的前一个快照）对应的日志。考虑到仍有 *Follower* 可能仍需要，避免为了几条日志而发送快照，*braft* 并不会立即删除当前快照对应的日志，而是等到下一次快照生成时，再删除本次快照对应的日志：

```cpp
```

> **互斥的快照任务**
>
> * 当有一个创建快照任务在执行时，即使快照超时
> * 快照任务本身没有超时时间，用户的快照任务可能一直卡在那里，而不知道的情况
>

> **快照的命名**
>

其他：创建快照失败
===

用户可以在 `on_snapshot_save` 中返回失败
