流程详解
===

快照

1. 当节点重启快照后，会进行加载快照来恢复状态机
2. 打开本地快照目录，返回 `SnapshotReader`
3. 将 `SnapshotReader` 作为参数回调用户状态机 `on_snapshot_load`
4. 等待快照加载完成
5. 若快照元数据中有集群配置，则回调用户状态机的 `on_configuration_committed`
6. 更新 `ApplyIndex` 为快照元数据中的 `lastIncludedIndex`
7. 若快照元数据中有集群配置，将则其设置为当前集群配置

流程概览
---

1. 当节点重启会从 Leader 下载完快照后，会触发加载快照的流程
2.

流程注解
---

由于快照中对应的是已经 committed 的日志，所以只要有快照，都可以直接加载

相关接口
---

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

节点重启需要遍历用户指定的快照目录，获取最新的快照目录

场景 1：节点重启
---

```cpp
int NodeImpl::init(const NodeOptions& options) {
    _options = options;

    // check _server_id
    if (butil::IP_ANY == _server_id.addr.ip) {
        LOG(ERROR) << "Group " << _group_id
                   << " Node can't started from IP_ANY";
        return -1;
    }

    if (!global_node_manager->server_exists(_server_id.addr)) {
        LOG(ERROR) << "Group " << _group_id
                   << " No RPC Server attached to " << _server_id.addr
                   << ", did you forget to call braft::add_service()?";
        return -1;
    }
    if (options.witness) {
        // When this node is a witness, set the election_timeout to be twice
        // of the normal replica to ensure that the normal replica has a higher
        // priority and is selected as the master
        if (FLAGS_raft_enable_witness_to_leader) {
            CHECK_EQ(0, _election_timer.init(this, options.election_timeout_ms * 2));
            CHECK_EQ(0, _vote_timer.init(this, options.election_timeout_ms * 2 + options.max_clock_drift_ms));
        }
    } else {
        CHECK_EQ(0, _election_timer.init(this, options.election_timeout_ms));
        CHECK_EQ(0, _vote_timer.init(this, options.election_timeout_ms + options.max_clock_drift_ms));
    }
    CHECK_EQ(0, _stepdown_timer.init(this, options.election_timeout_ms));
    CHECK_EQ(0, _snapshot_timer.init(this, options.snapshot_interval_s * 1000));

    _config_manager = new ConfigurationManager();

    if (bthread::execution_queue_start(&_apply_queue_id, NULL,
                                       execute_applying_tasks, this) != 0) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " fail to start execution_queue";
        return -1;
    }

    _apply_queue = execution_queue_address(_apply_queue_id);
    if (!_apply_queue) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " fail to address execution_queue";
        return -1;
    }

    // Create _fsm_caller first as log_manager needs it to report error
    _fsm_caller = new FSMCaller();

    _leader_lease.init(options.election_timeout_ms);
    if (options.witness) {
        _follower_lease.init(options.election_timeout_ms * 2, options.max_clock_drift_ms);
    } else {
        _follower_lease.init(options.election_timeout_ms, options.max_clock_drift_ms);
    }

    // log storage and log manager init
    if (init_log_storage() != 0) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " init_log_storage failed";
        return -1;
    }

    if (init_fsm_caller(LogId(0, 0)) != 0) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " init_fsm_caller failed";
        return -1;
    }

    // commitment manager init
    _ballot_box = new BallotBox();
    BallotBoxOptions ballot_box_options;
    ballot_box_options.waiter = _fsm_caller;
    ballot_box_options.closure_queue = _closure_queue;
    if (_ballot_box->init(ballot_box_options) != 0) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " init _ballot_box failed";
        return -1;
    }

    // snapshot storage init and load
    // NOTE: snapshot maybe discard entries when snapshot saved but not discard entries.
    //      init log storage before snapshot storage, snapshot storage will update configration
    if (init_snapshot_storage() != 0) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " init_snapshot_storage failed";
        return -1;
    }

    //
    butil::Status st = _log_manager->check_consistency();
    if (!st.ok()) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " is initialized with inconsitency log: "
                   << st;
        return -1;
    }

    _conf.id = LogId();
    // if have log using conf in log, else using conf in options
    if (_log_manager->last_log_index() > 0) {
        _log_manager->check_and_set_configuration(&_conf);
    } else {
        _conf.conf = _options.initial_conf;
    }

    // init meta and check term
    if (init_meta_storage() != 0) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " init_meta_storage failed";
        return -1;
    }

    // first start, we can vote directly
    if (_current_term == 1 && _voted_id.is_empty()) {
        _follower_lease.reset();
    }

    // init replicator
    ReplicatorGroupOptions rg_options;
    rg_options.heartbeat_timeout_ms = heartbeat_timeout(_options.election_timeout_ms);
    rg_options.election_timeout_ms = _options.election_timeout_ms;
    rg_options.log_manager = _log_manager;
    rg_options.ballot_box = _ballot_box;
    rg_options.node = this;
    rg_options.snapshot_throttle = _options.snapshot_throttle
        ? _options.snapshot_throttle->get()
        : NULL;
    rg_options.snapshot_storage = _snapshot_executor
        ? _snapshot_executor->snapshot_storage()
        : NULL;
    _replicator_group.init(NodeId(_group_id, _server_id), rg_options);

    // set state to follower
    _state = STATE_FOLLOWER;

    LOG(INFO) << "node " << _group_id << ":" << _server_id << " init,"
              << " term: " << _current_term
              << " last_log_id: " << _log_manager->last_log_id()
              << " conf: " << _conf.conf
              << " old_conf: " << _conf.old_conf;

    // start snapshot timer
    if (_snapshot_executor && _options.snapshot_interval_s > 0) {
        BRAFT_VLOG << "node " << _group_id << ":" << _server_id
                   << " term " << _current_term << " start snapshot_timer";
        _snapshot_timer.start();
    }

    if (!_conf.empty()) {
        step_down(_current_term, false, butil::Status::OK());
    }

    // add node to NodeManager
    if (!global_node_manager->add(this)) {
        LOG(ERROR) << "NodeManager add " << _group_id
                   << ":" << _server_id << " failed";
        return -1;
    }

    // Now the raft node is started , have to acquire the lock to avoid race
    // conditions
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (_conf.stable() && _conf.conf.size() == 1u
            && _conf.conf.contains(_server_id)) {
        // The group contains only this server which must be the LEADER, trigger
        // the timer immediately.
        elect_self(&lck);
    }

    return 0;
}
```

```cpp
int NodeImpl::init_snapshot_storage() {
    if (_options.snapshot_uri.empty()) {
        return 0;
    }
    _snapshot_executor = new SnapshotExecutor;
    SnapshotExecutorOptions opt;
    opt.uri = _options.snapshot_uri;
    opt.fsm_caller = _fsm_caller;
    opt.node = this;
    opt.log_manager = _log_manager;
    opt.addr = _server_id.addr;
    opt.init_term = _current_term;
    opt.filter_before_copy_remote = _options.filter_before_copy_remote;
    opt.usercode_in_pthread = _options.usercode_in_pthread;
    // not need to copy data file when it is witness.
    if (_options.witness) {
        opt.copy_file = false;
    }
    if (_options.snapshot_file_system_adaptor) {
        opt.file_system_adaptor = *_options.snapshot_file_system_adaptor;
    }
    // get snapshot_throttle
    if (_options.snapshot_throttle) {
        opt.snapshot_throttle = *_options.snapshot_throttle;
    }
    return _snapshot_executor->init(opt);
}
```

```cpp
int SnapshotExecutor::init(const SnapshotExecutorOptions& options) {
    if (options.uri.empty()) {
        LOG(ERROR) << "node " << _node->node_id() << " uri is empty()";
        return -1;
    }
    _log_manager = options.log_manager;
    _fsm_caller = options.fsm_caller;
    _node = options.node;
    _term = options.init_term;
    _usercode_in_pthread = options.usercode_in_pthread;

    _snapshot_storage = SnapshotStorage::create(options.uri);
    if (!_snapshot_storage) {
        LOG(ERROR)  << "node " << _node->node_id()
                    << " fail to find snapshot storage, uri " << options.uri;
        return -1;
    }
    if (options.filter_before_copy_remote) {
        _snapshot_storage->set_filter_before_copy_remote();
    }
    if (options.file_system_adaptor) {
        _snapshot_storage->set_file_system_adaptor(options.file_system_adaptor);
    }
    if (options.snapshot_throttle) {
        _snapshot_throttle = options.snapshot_throttle;
        _snapshot_storage->set_snapshot_throttle(options.snapshot_throttle);
    }
    if (_snapshot_storage->init() != 0) {
        LOG(ERROR) << "node " << _node->node_id()
                   << " fail to init snapshot storage, uri " << options.uri;
        return -1;
    }
    LocalSnapshotStorage* tmp = dynamic_cast<LocalSnapshotStorage*>(_snapshot_storage);
    if (tmp != NULL && !tmp->has_server_addr()) {
        tmp->set_server_addr(options.addr);
    }
    if (!options.copy_file) {
        tmp->set_copy_file(false);
    }
    SnapshotReader* reader = _snapshot_storage->open();
    if (reader == NULL) {
        return 0;
    }
    if (reader->load_meta(&_loading_snapshot_meta) != 0) {
        LOG(ERROR) << "Fail to load meta from `" << options.uri << "'";
        _snapshot_storage->close(reader);
        return -1;
    }
    _loading_snapshot = true;
    _running_jobs.add_count(1);
    // Load snapshot ater startup
    FirstSnapshotLoadDone done(this, reader);
    CHECK_EQ(0, _fsm_caller->on_snapshot_load(&done));
    done.wait_for_run();
    _snapshot_storage->close(reader);
    if (!done.status().ok()) {
        LOG(ERROR) << "Fail to load snapshot from " << options.uri;
        return -1;
    }
    return 0;
}
```

```cpp
int LocalSnapshotStorage::init() {
    butil::File::Error e;
    if (_fs == NULL) {
        _fs = default_file_system();
    }
    if (!_fs->create_directory(
                _path, &e, FLAGS_raft_create_parent_directories)) {
        LOG(ERROR) << "Fail to create " << _path << " : " << e;
        return -1;
    }
    // delete temp snapshot
    if (!_filter_before_copy_remote) {
        std::string temp_snapshot_path(_path);
        temp_snapshot_path.append("/");
        temp_snapshot_path.append(_s_temp_path);
        LOG(INFO) << "Deleting " << temp_snapshot_path;
        if (!_fs->delete_file(temp_snapshot_path, true)) {
            LOG(WARNING) << "delete temp snapshot path failed, path " << temp_snapshot_path;
            return EIO;
        }
    }

    // delete old snapshot
    DirReader* dir_reader = _fs->directory_reader(_path);
    if (!dir_reader->is_valid()) {
        LOG(WARNING) << "directory reader failed, maybe NOEXIST or PERMISSION. path: " << _path;
        delete dir_reader;
        return EIO;
    }
    std::set<int64_t> snapshots;
    while (dir_reader->next()) {
        int64_t index = 0;
        int match = sscanf(dir_reader->name(), BRAFT_SNAPSHOT_PATTERN, &index);
        if (match == 1) {
            snapshots.insert(index);
        }
    }
    delete dir_reader;

    // TODO: add snapshot watcher

    // get last_snapshot_index
    if (snapshots.size() > 0) {
        size_t snapshot_count = snapshots.size();
        for (size_t i = 0; i < snapshot_count - 1; i++) {
            int64_t index = *snapshots.begin();
            snapshots.erase(index);

            std::string snapshot_path(_path);
        butil::string_appendf(&snapshot_path, "/" BRAFT_SNAPSHOT_PATTERN, index);
            LOG(INFO) << "Deleting snapshot `" << snapshot_path << "'";
            // TODO: Notify Watcher before delete directories.
            if (!_fs->delete_file(snapshot_path, true)) {
                LOG(WARNING) << "delete old snapshot path failed, path " << snapshot_path;
                return EIO;
            }
        }

        _last_snapshot_index = *snapshots.begin();
        ref(_last_snapshot_index);
    }

    return 0;
}
```

场景 2：安装快照
---

```cpp
void SnapshotExecutor::load_downloading_snapshot(DownloadingSnapshot* ds,
                                                 const SnapshotMeta& meta) {
    std::unique_ptr<DownloadingSnapshot> ds_guard(ds);
    std::unique_lock<raft_mutex_t> lck(_mutex);
    CHECK_EQ(ds, _downloading_snapshot.load(butil::memory_order_relaxed));
    brpc::ClosureGuard done_guard(ds->done);
    CHECK(_cur_copier);
    SnapshotReader* reader = _cur_copier->get_reader();
    if (!_cur_copier->ok()) {
        if (_cur_copier->error_code() == EIO) {
            report_error(_cur_copier->error_code(),
                         "%s", _cur_copier->error_cstr());
        }
        if (reader) {
            _snapshot_storage->close(reader);
        }
        ds->cntl->SetFailed(_cur_copier->error_code(), "%s",
                            _cur_copier->error_cstr());
        _snapshot_storage->close(_cur_copier);
        _cur_copier = NULL;
        _downloading_snapshot.store(NULL, butil::memory_order_relaxed);
        // Release the lock before responding the RPC
        lck.unlock();
        _running_jobs.signal();
        return;
    }
    _snapshot_storage->close(_cur_copier);
    _cur_copier = NULL;
    if (reader == NULL || !reader->ok()) {
        if (reader) {
            _snapshot_storage->close(reader);
        }
        _downloading_snapshot.store(NULL, butil::memory_order_release);
        lck.unlock();
        ds->cntl->SetFailed(brpc::EINTERNAL,
                           "Fail to copy snapshot from %s",
                            ds->request->uri().c_str());
        _running_jobs.signal();
        return;
    }
    // The owner of ds is on_snapshot_load_done
    ds_guard.release();
    done_guard.release();
    _loading_snapshot = true;
    //                ^ After this point, this installing cannot be interrupted
    _loading_snapshot_meta = meta;
    lck.unlock();
    InstallSnapshotDone* install_snapshot_done =
            new InstallSnapshotDone(this, reader);
    int ret = _fsm_caller->on_snapshot_load(install_snapshot_done);
    if (ret != 0) {
        LOG(WARNING) << "node " << _node->node_id() << " fail to call on_snapshot_load";
        install_snapshot_done->status().set_error(EHOSTDOWN, "This raft node is down");
        return install_snapshot_done->Run();
    }
}
```

阶段二：用户加载快照
===

```cpp
int FSMCaller::on_snapshot_load(LoadSnapshotClosure* done) {
    ApplyTask task;
    task.type = SNAPSHOT_LOAD;
    task.done = done;
    return bthread::execution_queue_execute(_queue_id, task);
}

void FSMCaller::do_snapshot_load(LoadSnapshotClosure* done) {
    //TODO done_guard
    SnapshotReader* reader = done->start();
    if (!reader) {
        done->status().set_error(EINVAL, "open SnapshotReader failed");
        done->Run();
        return;
    }

    SnapshotMeta meta;
    int ret = reader->load_meta(&meta);
    if (0 != ret) {
        done->status().set_error(ret, "SnapshotReader load_meta failed.");
        done->Run();
        if (ret == EIO) {
            Error e;
            e.set_type(ERROR_TYPE_SNAPSHOT);
            e.status().set_error(ret, "Fail to load snapshot meta");
            set_error(e);
        }
        return;
    }

    LogId last_applied_id;
    last_applied_id.index = _last_applied_index.load(butil::memory_order_relaxed);
    last_applied_id.term = _last_applied_term;
    LogId snapshot_id;
    snapshot_id.index = meta.last_included_index();
    snapshot_id.term = meta.last_included_term();
    if (last_applied_id > snapshot_id) {
        done->status().set_error(ESTALE,"Loading a stale snapshot"
                                 " last_applied_index=%" PRId64 " last_applied_term=%" PRId64
                                 " snapshot_index=%" PRId64 " snapshot_term=%" PRId64,
                                 last_applied_id.index, last_applied_id.term,
                                 snapshot_id.index, snapshot_id.term);
        return done->Run();
    }

    ret = _fsm->on_snapshot_load(reader);
    if (ret != 0) {
        done->status().set_error(ret, "StateMachine on_snapshot_load failed");
        done->Run();
        Error e;
        e.set_type(ERROR_TYPE_STATE_MACHINE);
        e.status().set_error(ret, "StateMachine on_snapshot_load failed");
        set_error(e);
        return;
    }

    if (meta.old_peers_size() == 0) {
        // Joint stage is not supposed to be noticeable by end users.
        Configuration conf;
        for (int i = 0; i < meta.peers_size(); ++i) {
            conf.add_peer(meta.peers(i));
        }
        _fsm->on_configuration_committed(conf, meta.last_included_index());
    }

    _last_applied_index.store(meta.last_included_index(),
                              butil::memory_order_release);
    _last_applied_term = meta.last_included_term();
    done->Run();
}
```


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
