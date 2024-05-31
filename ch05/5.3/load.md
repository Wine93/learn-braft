流程详解
===

流程概览
---

当节点重启后或安装来自 Leader 的快照后，会进行加载快照来恢复状态机，其流程如下：
1. 节点会将加载快照任务放进 [ApplyTaskQueue][ApplyTaskQueue]，等待其被执行
2. 当任务被执行时，会打开本地快照目录，返回 `SnapshotReader`
3. 将 `SnapshotReader` 作为参数调用用户状态机的 `on_snapshot_load`
4. 节点等待快照加载完成（同步操作）
5. 根据快照元数据中的节点配置，调用用户状态机的 `on_configuration_committed`
6. 更新 `applyIndex` 为快照元数据中的 `lastIncludedIndex`
7. 若快照元数据中有集群配置，将则其设置为当前集群配置？若？

[ApplyTaskQueue]: /ch02/2.1/init.md#applytaskqueue

相关接口
---

加载快照时会调用的状态机函数：

```cpp
class StateMachine {
public:
    // user defined snapshot load function
    // get and load snapshot
    // success return 0, fail return errno
    // Default: Load nothing and returns error.
    virtual int on_snapshot_load(::braft::SnapshotReader* reader);

    // Invoked when a configuration has been committed to the group
    virtual void on_configuration_committed(const ::braft::Configuration& conf);
    virtual void on_configuration_committed(const ::braft::Configuration& conf, int64_t index);
};
```

读取快照的 `SnapshotReader`：

```cpp
class SnapshotReader : public Snapshot {
public:
    // Load meta from
    virtual int load_meta(SnapshotMeta* meta) = 0;

    // Generate uri for other peers to copy this snapshot.
    // Return an empty string if some error has occcured
    virtual std::string generate_uri_for_copy() = 0;
};

class Snapshot : public butil::Status {
public:
    // Get the path of the Snapshot
    virtual std::string get_path() = 0;

    // List all the existing files in the Snapshot currently
    virtual void list_files(std::vector<std::string> *files) = 0;

    // Get the implementation-defined file_meta
    virtual int get_file_meta(const std::string& filename,
                              ::google::protobuf::Message* file_meta) {
        (void)filename;
        if (file_meta != NULL) {
            file_meta->Clear();
        }
        return 0;
    }
};
```

阶段一：触发加载快照
===

触发加载快照
---

节点在重启或安装完 Leader 的快照后，会触发加载快照。这两种场景都会调用 `FSMCaller::on_snapshot_load` 加载快照，只不过加载快照完成后的回调函数不同：

* 重启：`FirstSnapshotLoadDone`
* 安装快照：`InstallSnapshotDone`

**场景 1：节点重启**

节点重启时会遍历快照存储目录，获取最新的快照目录，将其打开并返回 `SnapshotReader`，然后调用 `FSMCaller::on_snapshot_load` 加载快照，并同步等待其加载完成：

```cpp
// (1) 重启时，调用 `init_snapshot_storage` 加载快照
int NodeImpl::init(const NodeOptions& options) {
    ...
    // snapshot storage init and load
    if (init_snapshot_storage() != 0) {
        ...
        return -1;
    }
    ..
    return 0;
}

// (2) `init_snapshot_storage` 会调用 `SnapshotExecutor::init`
int NodeImpl::init_snapshot_storage() {
    ...
    return _snapshot_executor->init(opt);
}

int SnapshotExecutor::init(const SnapshotExecutorOptions& options) {
    ...
    // (3) 首先会打开快照存储目录，遍历该目录下的所有快照目录，
    //     根据快照目录名（以 applyIndex 命名）找到最新的快照
    if (_snapshot_storage->init() != 0) {
        ...
        return -1;
    }
    // (4) 打开快照目录，返回 SnapshotReader
    ...
    SnapshotReader* reader = _snapshot_storage->open();
    ...
    // (5) 生成快照加载完成后的回调函数
    FirstSnapshotLoadDone done(this, reader);
    // (6) 调用 `FSMCaller::on_snapshot_load` 加载快照
    CHECK_EQ(0, _fsm_caller->on_snapshot_load(&done));
    // (7) 等待快照加载完毕
    done.wait_for_run();
    ...
    return 0;
}
```

**场景 2：安装快照**

当节点下载完 Leader 的快照时，会调用 `load_downloading_snapshot` 加载快照：

```cpp
void SnapshotExecutor::load_downloading_snapshot(DownloadingSnapshot* ds,
                                                 const SnapshotMeta& meta) {
    ...
    // (1) 打开从 Leader 下载的快照目录，返回 `SnapshotReader`
    SnapshotReader* reader = _cur_copier->get_reader();
    ...
    _snapshot_storage->close(_cur_copier);
    ...
    // (2) 生成快照加载完毕后的回调函数
    InstallSnapshotDone* install_snapshot_done =
            new InstallSnapshotDone(this, reader);
    // (6) 调用 `FSMCaller::on_snapshot_load` 加载快照
    int ret = _fsm_caller->on_snapshot_load(install_snapshot_done);
    ...
}
```

任务入队
---

`on_snapshot_load` 会将加载快照任务放进 [ApplyTaskQueue][ApplyTaskQueue]，等待其被执行：

```cpp
int FSMCaller::on_snapshot_load(LoadSnapshotClosure* done) {
    ApplyTask task;
    task.type = SNAPSHOT_LOAD;
    task.done = done;
    return bthread::execution_queue_execute(_queue_id, task);
}
```

执行加载快照
---

队列的消费函数 `run` 会调用 `FSMCaller::do_snapshot_load` 执加载快照：

```cpp
int FSMCaller::run(void* meta, bthread::TaskIterator<ApplyTask>& iter) {
    ...
    for (; iter; ++iter) {
        ...
        switch (iter->type) {
        ...
        case SNAPSHOT_LOAD:
            caller->_cur_task = SNAPSHOT_LOAD;
            ...
            if (caller->pass_by_status(iter->done)) {
                caller->do_snapshot_load((LoadSnapshotClosure*)iter->done);
            }
            break;
        ...
        };
    }
    ...
    return 0;
}
```

`do_snapshot_load` 函数会做以下几件事：

```cpp
void FSMCaller::do_snapshot_load(LoadSnapshotClosure* done) {
    ...
    SnapshotReader* reader = done->start();

    // (1) 获取快照的元数据
    SnapshotMeta meta;
    int ret = reader->load_meta(&meta);
    if (0 != ret) {
        ...
        return;
    }

    // (2) 调用用户状态机的 `on_snapshot_load` 加载快照
    ret = _fsm->on_snapshot_load(reader);
    if (ret != 0) {
        done->status().set_error(ret, "StateMachine on_snapshot_load failed");
        done->Run();
        ...
        return;
    }

    // (3) 获取快照元数据中的节点配置，
    //     并以该配置调用用户状态机的 `on_configuration_committed`
    if (meta.old_peers_size() == 0) {
        // Joint stage is not supposed to be noticeable by end users.
        Configuration conf;
        for (int i = 0; i < meta.peers_size(); ++i) {
            conf.add_peer(meta.peers(i));
        }
        _fsm->on_configuration_committed(conf, meta.last_included_index());
    }

    // (4) 设置 applyIndex 为快照元数据中的 lastIncludeIndex
    _last_applied_index.store(meta.last_included_index(),
                              butil::memory_order_release);
    _last_applied_term = meta.last_included_term();

    done->Run();
}
```

阶段二：用户加载快照
===

on_snapshot_load
---

用户需要实现状态机的 `on_snapshot_load` 函数来加载快照：

```cpp
class StateMachine {
public:
    // user defined snapshot load function
    // get and load snapshot
    // success return 0, fail return errno
    // Default: Load nothing and returns error.
    virtual int on_snapshot_load(::braft::SnapshotReader* reader);
};
```

get_path
---

load_meta
---

list_files
---


阶段三：完成快照加载
===

```cpp
class FirstSnapshotLoadDone : public LoadSnapshotClosure {
public:
    ...
    void Run() {
        _se->on_snapshot_load_done(status());  // _se: SnapshotExecutor
    }
    ...
};

void InstallSnapshotDone::Run() {
    _se->on_snapshot_load_done(status());
    delete this;
}
```

```cpp
void SnapshotExecutor::on_snapshot_load_done(const butil::Status& st) {
    std::unique_lock<raft_mutex_t> lck(_mutex);

    CHECK(_loading_snapshot);
    DownloadingSnapshot* m = _downloading_snapshot.load(butil::memory_order_relaxed);

    if (st.ok()) {
        _last_snapshot_index = _loading_snapshot_meta.last_included_index();
        _last_snapshot_term = _loading_snapshot_meta.last_included_term();
        _log_manager->set_snapshot(&_loading_snapshot_meta);
    }
    std::stringstream ss;
    if (_node) {
        ss << "node " << _node->node_id() << ' ';
    }
    ss << "snapshot_load_done, "
              << _loading_snapshot_meta.ShortDebugString();
    LOG(INFO) << ss.str();
    lck.unlock();
    if (_node) {
        // FIXME: race with set_peer, not sure if this is fine
        _node->update_configuration_after_installing_snapshot();
    }
    lck.lock();
    _loading_snapshot = false;
    _downloading_snapshot.store(NULL, butil::memory_order_release);
    lck.unlock();
    if (m) {
        // Respond RPC
        if (!st.ok()) {
            m->cntl->SetFailed(st.error_code(), "%s", st.error_cstr());
        } else {
            m->response->set_success(true);
        }
        m->done->Run();
        delete m;
    }
    _running_jobs.signal();
}
```

其他：加载失败
===
