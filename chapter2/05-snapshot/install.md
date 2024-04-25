简介
===

```cpp
快照对应的日志会不会重放？ // 应该没有被删除
```

整体流程
===

步骤一：Leader 向 Follower 发送安装快照指令
---

步骤二：Follower 从 Leader 下载快照文件
---

步骤三: Follower 加载快照
---

加载的时候是不是不再接受 Leader 的日志? 或者 Leader 不再发送日志

>
> 保存快照：
>
> 安装快照：*Follower*
>
> 加载快照：
>
> 对于任一节点来说，这 3 类任务是互斥的，当有
>

具体实现
===

加载快照
---

```cpp
void SnapshotExecutor::load_downloading_snapshot(DownloadingSnapshot* ds,
                                                 const SnapshotMeta& meta) {
    _snapshot_storage->close(_cur_copier);
    ...
    InstallSnapshotDone* install_snapshot_done =
            new InstallSnapshotDone(this, reader);
    int ret = _fsm_caller->on_snapshot_load(install_snapshot_done);
}
```

步骤二：从 Leader 下载快照文件
---

```proto
message GetFileRequest {
    required int64 reader_id = 1;
    required string filename = 2;
    required int64 count = 3;
    required int64 offset = 4;
    optional bool read_partly = 5;
}

message GetFileResponse {
    // Data is in attachment
    required bool eof = 1;
    optional int64 read_size = 2;
}

service FileService {
    rpc get_file(GetFileRequest) returns (GetFileResponse);
}
```

```proto
message SnapshotMeta {
    required int64 last_included_index = 1;
    required int64 last_included_term = 2;
    repeated string peers = 3;
    repeated string old_peers = 4;
}

message InstallSnapshotRequest {
    required string group_id = 1;
    required string server_id = 2;
    required string peer_id = 3;
    required int64 term = 4;
    required SnapshotMeta meta = 5;
    required string uri = 6;  // remote://ip:port/reader_id
};

message InstallSnapshotResponse {
    required int64 term = 1;
    required bool success = 2;
};
```

```cpp
void RaftServiceImpl::install_snapshot(google::protobuf::RpcController* cntl_base,
                              const InstallSnapshotRequest* request,
                              InstallSnapshotResponse* response,
                              google::protobuf::Closure* done) {
    ...
    node->handle_install_snapshot_request(cntl, request, response, done);
}
```

```cpp
void NodeImpl::handle_install_snapshot_request(brpc::Controller* cntl,
                                    const InstallSnapshotRequest* request,
                                    InstallSnapshotResponse* response,
                                    google::protobuf::Closure* done) {
    ...
    clear_append_entries_cache();
    ...
    return _snapshot_executor->install_snapshot(
            cntl, request, response, done_guard.release());
}
```

```cpp
void SnapshotExecutor::install_snapshot(brpc::Controller* cntl,
                                        const InstallSnapshotRequest* request,
                                        InstallSnapshotResponse* response,
                                        google::protobuf::Closure* done) {
    ...
    return load_downloading_snapshot(ds.release(), meta);
}
```

```cpp
int SnapshotExecutor::register_downloading_snapshot(DownloadingSnapshot* ds) {
    ...
    if (_saving_snapshot) {
        LOG(WARNING) << "Register failed: is saving snapshot.";
        ds->cntl->SetFailed(EBUSY, "Is saving snapshot");
        return -1;
    }

    ...

    DownloadingSnapshot* m = _downloading_snapshot.load(
            butil::memory_order_relaxed);
    if (!m) {
        _downloading_snapshot.store(ds, butil::memory_order_relaxed);
        _cur_copier = _snapshot_storage->start_to_copy_from(ds->request->uri());
    }
}
```

```cpp
SnapshotCopier* LocalSnapshotStorage::start_to_copy_from(const std::string& uri) {
    if (copier->init(uri) != 0) {  // copier: LocalSnapshotCopier
    }

    copier->start();
    return copier;
}
```

```cpp
int LocalSnapshotCopier::init(const std::string& uri) {
    return _copier.init(uri, _fs, _throttle);  // _copier: RemoteFileCopier
}
```

```cpp
int RemoteFileCopier::init(const std::string& uri, FileSystemAdaptor* fs,
        SnapshotThrottle* throttle) {
}
```

```cpp
void LocalSnapshotCopier::start() {
    if (bthread_start_background(
                &_tid, NULL, start_copy, this) != 0) {
        PLOG(ERROR) << "Fail to start bthread";
        copy();
    }
}
```

```cpp
void LocalSnapshotCopier::copy() {
    do {
        load_meta_table();
        if (!ok()) {
            break;
        }
        filter();
        if (!ok()) {
            break;
        }
        if (!_copy_file) {
            break;
        }
        std::vector<std::string> files;
        _remote_snapshot.list_files(&files);
        for (size_t i = 0; i < files.size() && ok(); ++i) {
            copy_file(files[i]);
        }
    } while (0);
    if (!ok() && _writer && _writer->ok()) {
        LOG(WARNING) << "Fail to copy, error_code " << error_code()
                     << " error_msg " << error_cstr()
                     << " writer path " << _writer->get_path();
        _writer->set_error(error_code(), error_cstr());
    }
    if (_writer) {
        // set_error for copier only when failed to close writer and copier was
        // ok before this moment
        if (_storage->close(_writer, _filter_before_copy_remote) != 0 && ok()) {
            set_error(EIO, "Fail to close writer");
        }
        _writer = NULL;
    }
    if (ok()) {
        _reader = _storage->open();
    }
}
```



收尾工作
---

```cpp
void InstallSnapshotDone::Run() {
    _se->on_snapshot_load_done(status());  // _se: SnapshotExecutor
    delete this;
}

void SnapshotExecutor::on_snapshot_load_done(const butil::Status& st) {
    if (st.ok()) {
        _last_snapshot_index = _loading_snapshot_meta.last_included_index();
        _last_snapshot_term = _loading_snapshot_meta.last_included_term();
        _log_manager->set_snapshot(&_loading_snapshot_meta);
    }

    ...
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
}
```

```cpp
void LogManager::set_snapshot(const SnapshotMeta* meta) {
    ...
    _last_snapshot_id.index = meta->last_included_index();
    _last_snapshot_id.term = meta->last_included_term();
    if (_last_snapshot_id > _applied_id) {
        _applied_id = _last_snapshot_id;
    }
}
```

```cpp
int LocalSnapshotStorage::close(SnapshotWriter* writer_base,
                                bool keep_data_on_error) {

     do {
        ret = writer->sync();

        ...

        // rename temp to new
        std::string temp_path(_path);
        temp_path.append("/");
        temp_path.append(_s_temp_path);
        std::string new_path(_path);
        butil::string_appendf(&new_path, "/" BRAFT_SNAPSHOT_PATTERN, new_index);

        if (!_fs->delete_file(new_path, true)) {
            ...
        }

        if (!_fs->rename(temp_path, new_path)) {
            ...
        }

        ...
        ref(new_index);
        {
            BAIDU_SCOPED_LOCK(_mutex);
            CHECK_EQ(old_index, _last_snapshot_index);
            _last_snapshot_index = new_index;
        }
        // unref old_index, ref new_index
        unref(old_index);
    } while (0);
}
```

```cpp
void LocalSnapshotStorage::unref(const int64_t index) {
    std::unique_lock<raft_mutex_t> lck(_mutex);
    std::map<int64_t, int>::iterator it = _ref_map.find(index);
    if (it != _ref_map.end()) {
        it->second--;

        if (it->second == 0) {
            _ref_map.erase(it);
            lck.unlock();
            std::string old_path(_path);
            butil::string_appendf(&old_path, "/" BRAFT_SNAPSHOT_PATTERN, index);
            destroy_snapshot(old_path);
        }
    }
}
```

```cpp
int LocalSnapshotStorage::destroy_snapshot(const std::string& path) {
    LOG(INFO) << "Deleting "  << path;
    if (!_fs->delete_file(path, true)) {
        LOG(WARNING) << "delete old snapshot path failed, path " << path;
        return -1;
    }
    return 0;
}
```
