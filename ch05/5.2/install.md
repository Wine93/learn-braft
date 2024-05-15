流程详解
===

流程概览
---

1. 当 Leader 已经压缩了 Follower 需要复制的日志，Leader 就需要将快照发送给 Follower
2. Leader 向 Follower 发送 `InstallSnapshot` 请求
3. Follower 根据请求中的 `URI` 分批次向 Leader 下载快照对应的文件集：
   * 3.1 创建连接 Leader 的客户端
   * 3.2 发送 `GetFileRequest` 请求获取 Leader 快照的元数据
   * 3.3 创建 `temp` 目录保存用于保存下载的临时快照
   * 3.3 根据元数据中的文件列表对比本地快照与远程快照的差异，获得下载文件列表
   * 3.4 根据文件列表，逐一发送 `GetFileRequest` 请求获得文件保存至临时快照中
   * 3.5 待快照下载完成后，删除本地快照，并将临时快照 `rename()` 成正式快照
4. Follower 回调用户状态机的 `on_snapshot_load` 加载快照
5. 等待快照加载完毕后：
    * 5.1 更新 ApplyIndex 为快照的 `lastIncludedIndex`
    * 5.2 删除 `Index` 小于 `lastIncludedIndex` 的日志
6. Follower 向 Leader 发送成功的 `InstallSnapshot` 响应
7. Leader 收到成功响应后更新 Follower 的 `nextIndex` 为快照的 `lastIncludedIndex` + 1
8. Leader 从 `nextIndex` 开始继续向 Follower 发送日志

流程注解
---

1. 正常情况下不会触发快照，例如节点故障下线后重启

增量下载
---


相关 RPC
---

`InstallSnapshot`：

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
    required string uri = 6;
};

message InstallSnapshotResponse {
    required int64 term = 1;
    required bool success = 2;
};

service RaftService {
    rpc install_snapshot(InstallSnapshotRequest) returns (InstallSnapshotResponse);
};
```

下载文件：
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

相关接口
---

```cpp
```



```cpp
```

阶段一：Leader 下发命令
===

Follower 处理 `InstallSnapshot`
---

```cpp
void RaftServiceImpl::install_snapshot(google::protobuf::RpcController* cntl_base,
                              const InstallSnapshotRequest* request,
                              InstallSnapshotResponse* response,
                              google::protobuf::Closure* done) {
    ...
    node->handle_install_snapshot_request(cntl, request, response, done);
}

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

void SnapshotExecutor::install_snapshot(brpc::Controller* cntl,
                                        const InstallSnapshotRequest* request,
                                        InstallSnapshotResponse* response,
                                        google::protobuf::Closure* done) {
    ...
    std::unique_ptr<DownloadingSnapshot> ds(new DownloadingSnapshot);
    ...
    ret = register_downloading_snapshot(ds.get());
    ...
    _cur_copier->join();

    ...
    return load_downloading_snapshot(ds.release(), meta);
}
```

阶段二：Follower 下载快照
===


创建下载快照的任务：
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
        // _cur_copier: LocalSnapshotCopier
        _cur_copier = _snapshot_storage->start_to_copy_from(ds->request->uri());
    }
}
```

```cpp
SnapshotCopier* LocalSnapshotStorage::start_to_copy_from(const std::string& uri) {
    LocalSnapshotCopier* copier = new LocalSnapshotCopier(_copy_file);
    if (copier->init(uri) != 0) {  // copier: LocalSnapshotCopier
    }

    copier->start();
    return copier;
}
```


初始化客户端
---

初始化下载文件的客户端 `RemoteFileCopier`：
```cpp
int LocalSnapshotCopier::init(const std::string& uri) {
    return _copier.init(uri, _fs, _throttle);  // _copier: RemoteFileCopier
}

int RemoteFileCopier::init(const std::string& uri, FileSystemAdaptor* fs,
        SnapshotThrottle* throttle) {
    // Parse uri format: remote://ip:port/reader_id
    static const size_t prefix_size = strlen("remote://");
    butil::StringPiece uri_str(uri);
    if (!uri_str.starts_with("remote://")) {
        LOG(ERROR) << "Invalid uri=" << uri;
        return -1;
    }
    uri_str.remove_prefix(prefix_size);
    size_t slash_pos = uri_str.find('/');
    butil::StringPiece ip_and_port = uri_str.substr(0, slash_pos);
    uri_str.remove_prefix(slash_pos + 1);
    if (!butil::StringToInt64(uri_str, &_reader_id)) {
        LOG(ERROR) << "Invalid reader_id_format=" << uri_str
                   << " in " << uri;
        return -1;
    }
    brpc::ChannelOptions channel_opt;
    channel_opt.connect_timeout_ms = FLAGS_raft_rpc_channel_connect_timeout_ms;
    if (_channel.Init(ip_and_port.as_string().c_str(), &channel_opt) != 0) {
        LOG(ERROR) << "Fail to init Channel to " << ip_and_port;
        return -1;
    }
    _fs = fs;
    _throttle = throttle;
    return 0;
}
```

启动客户端
---
```cpp
void LocalSnapshotCopier::start() {
    if (bthread_start_background(
                &_tid, NULL, start_copy, this) != 0) {
        PLOG(ERROR) << "Fail to start bthread";
        copy();
    }
}

void *LocalSnapshotCopier::start_copy(void* arg) {
    LocalSnapshotCopier* c = (LocalSnapshotCopier*)arg;
    c->copy();
    return NULL;
}
```

Follower 下载流程
---

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

下载快照元数据
---

```cpp
void LocalSnapshotCopier::load_meta_table() {
    butil::IOBuf meta_buf;
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (_cancelled) {
        set_error(ECANCELED, "%s", berror(ECANCELED));
        return;
    }
    scoped_refptr<RemoteFileCopier::Session> session
            = _copier.start_to_copy_to_iobuf(BRAFT_SNAPSHOT_META_FILE,
                                            &meta_buf, NULL);
    _cur_session = session.get();
    lck.unlock();
    session->join();
    lck.lock();
    _cur_session = NULL;
    lck.unlock();
    if (!session->status().ok()) {
        LOG(WARNING) << "Fail to copy meta file : " << session->status();
        set_error(session->status().error_code(), session->status().error_cstr());
        return;
    }
    if (_remote_snapshot._meta_table.load_from_iobuf_as_remote(meta_buf) != 0) {
        LOG(WARNING) << "Bad meta_table format";
        set_error(-1, "Bad meta_table format");
        return;
    }
    CHECK(_remote_snapshot._meta_table.has_meta());
}
```

过滤文件列表
---

```cpp
void LocalSnapshotCopier::filter() {
    _writer = (LocalSnapshotWriter*)_storage->create(!_filter_before_copy_remote);
    if (_writer == NULL) {
        set_error(EIO, "Fail to create snapshot writer");
        return;
    }

    if (_filter_before_copy_remote) {  // true
        SnapshotReader* reader = _storage->open();
        if (filter_before_copy(_writer, reader) != 0) {
            LOG(WARNING) << "Fail to filter writer before copying"
                            ", path: " << _writer->get_path()
                         << ", destroy and create a new writer";
            _writer->set_error(-1, "Fail to filter");
            _storage->close(_writer, false);
            _writer = (LocalSnapshotWriter*)_storage->create(true);
        }
        if (reader) {
            _storage->close(reader);
        }
        if (_writer == NULL) {
            set_error(EIO, "Fail to create snapshot writer");
            return;
        }
    }
    _writer->save_meta(_remote_snapshot._meta_table.meta());
    if (_writer->sync() != 0) {
        set_error(EIO, "Fail to sync snapshot writer");
        return;
    }
}
```

`from_empty` 为 False，不删除 `temp` 目录
```cpp
SnapshotWriter* LocalSnapshotStorage::create(bool from_empty) {
    LocalSnapshotWriter* writer = NULL;

    do {
        std::string snapshot_path(_path);  // ./data/temp
        snapshot_path.append("/");
        snapshot_path.append(_s_temp_path);

        // delete temp
        // TODO: Notify watcher before deleting
        if (_fs->path_exists(snapshot_path) && from_empty) {
            if (destroy_snapshot(snapshot_path) != 0) {
                break;
            }
        }

        writer = new LocalSnapshotWriter(snapshot_path, _fs.get());
        if (writer->init() != 0) {
            LOG(ERROR) << "Fail to init writer in path " << snapshot_path
                       << ", " << *writer;
            delete writer;
            writer = NULL;
            break;
        }
        BRAFT_VLOG << "Create writer success, path: " << snapshot_path;
    } while (0);

    return writer;
}
```

遍历用户指定的快照目录，保存最近的一个快照，删除其余全部快照：

```cpp
int LocalSnapshotWriter::init() {
    butil::File::Error e;
    if (!_fs->create_directory(_path, &e, false)) {
        LOG(ERROR) << "Fail to create directory " << _path << ", " << e;
        set_error(EIO, "CreateDirectory failed with path: %s", _path.c_str());
        return EIO;
    }
    std::string meta_path = _path + "/" BRAFT_SNAPSHOT_META_FILE;
    if (_fs->path_exists(meta_path) &&
                _meta_table.load_from_file(_fs, meta_path) != 0) {
        LOG(ERROR) << "Fail to load meta from " << meta_path;
        set_error(EIO, "Fail to load metatable from %s", meta_path.c_str());
        return EIO;
    }

    // remove file if meta_path not exist or it's not in _meta_table
    // to avoid dirty data
    {
        std::vector<std::string> to_remove;
        DirReader* dir_reader = _fs->directory_reader(_path);
        if (!dir_reader->is_valid()) {
            LOG(ERROR) << "Invalid directory reader, maybe NOEXIST or PERMISSION,"
                       << " path: " << _path;
            set_error(EIO, "Invalid directory reader in path: %s", _path.c_str());
            delete dir_reader;
            return EIO;
        }
        while (dir_reader->next()) {
            std::string filename = dir_reader->name();
            if (filename != BRAFT_SNAPSHOT_META_FILE) {
                if (get_file_meta(filename, NULL) != 0) {
                    to_remove.push_back(filename);
                }
            }
        }
        delete dir_reader;
        for (size_t i = 0; i < to_remove.size(); ++i) {
            std::string file_path = _path + "/" + to_remove[i];
            _fs->delete_file(file_path, false);
            LOG(WARNING) << "Snapshot file exist but meta not found so delete it,"
                << " path: " << file_path;
        }
    }

    return 0;
}
```

```cpp
// write: temp 目录的 SnapshotWrite ? 自己打的快照还是下载的？
// last_snapshot: 最新的快照
int LocalSnapshotCopier::filter_before_copy(LocalSnapshotWriter* writer,
                                            SnapshotReader* last_snapshot) {
    std::vector<std::string> existing_files;
    writer->list_files(&existing_files);
    std::vector<std::string> to_remove;

    for (size_t i = 0; i < existing_files.size(); ++i) {
        if (_remote_snapshot.get_file_meta(existing_files[i], NULL) != 0) {
            to_remove.push_back(existing_files[i]);
            writer->remove_file(existing_files[i]);  // 将文件名从元数据表中移除
        }
    }

    std::vector<std::string> remote_files;
    _remote_snapshot.list_files(&remote_files);
    for (size_t i = 0; i < remote_files.size(); ++i) {
        const std::string& filename = remote_files[i];
        LocalFileMeta remote_meta;
        CHECK_EQ(0, _remote_snapshot.get_file_meta(
                filename, &remote_meta));
        if (!remote_meta.has_checksum()) {
            // Redownload file if this file doen't have checksum
            writer->remove_file(filename);
            to_remove.push_back(filename);
            continue;
        }

        LocalFileMeta local_meta;
        if (writer->get_file_meta(filename, &local_meta) == 0) {  // temp 目录有
            if (local_meta.has_checksum() &&
                local_meta.checksum() == remote_meta.checksum()) {
                LOG(INFO) << "Keep file=" << filename
                          << " checksum=" << remote_meta.checksum()
                          << " in " << writer->get_path();
                continue;
            }
            // Remove files from writer so that the file is to be copied from
            // remote_snapshot or last_snapshot
            writer->remove_file(filename);
            to_remove.push_back(filename);
        }

        // Try find files in last_snapshot
        if (!last_snapshot) {
            continue;
        }
        if (last_snapshot->get_file_meta(filename, &local_meta) != 0) {
            continue;
        }
        if (!local_meta.has_checksum() || local_meta.checksum() != remote_meta.checksum()) {
            continue;
        }
        LOG(INFO) << "Found the same file=" << filename
                  << " checksum=" << remote_meta.checksum()
                  << " in last_snapshot=" << last_snapshot->get_path();
        if (local_meta.source() == braft::FILE_SOURCE_LOCAL) {
            std::string source_path = last_snapshot->get_path() + '/'
                                      + filename;
            std::string dest_path = writer->get_path() + '/'
                                      + filename;
            _fs->delete_file(dest_path, false);
            if (!_fs->link(source_path, dest_path)) {
                PLOG(ERROR) << "Fail to link " << source_path
                            << " to " << dest_path;
                continue;
            }
            // Don't delete linked file
            if (!to_remove.empty() && to_remove.back() == filename) {
                to_remove.pop_back();
            }
        }
        // Copy file from last_snapshot
        writer->add_file(filename, &local_meta);
    }

    if (writer->sync() != 0) {
        LOG(ERROR) << "Fail to sync writer on path=" << writer->get_path();
        return -1;
    }

    for (size_t i = 0; i < to_remove.size(); ++i) {
        std::string file_path = writer->get_path() + "/" + to_remove[i];
        _fs->delete_file(file_path, false);
    }

    return 0;
}
```

逐一下载文件
---

```cpp

void LocalSnapshot::list_files(std::vector<std::string> *files) {
    return _meta_table.list_files(files);  // _meta_table: LocalSnapshotMetaTable
}

void LocalSnapshotMetaTable::list_files(std::vector<std::string>* files) const {
    if (!files) {
        return;
    }
    files->clear();
    files->reserve(_file_map.size());
    for (Map::const_iterator
            iter = _file_map.begin(); iter != _file_map.end(); ++iter) {
        files->push_back(iter->first);
    }
}
```

```cpp
void LocalSnapshotCopier::copy_file(const std::string& filename) {
    if (_writer->get_file_meta(filename, NULL) == 0) {
        LOG(INFO) << "Skipped downloading " << filename
                  << " path: " << _writer->get_path();
        return;
    }
    std::string file_path = _writer->get_path() + '/' + filename;
    butil::FilePath sub_path(filename);
    if (sub_path != sub_path.DirName() && sub_path.DirName().value() != ".") {
        butil::File::Error e;
        bool rc = false;
        if (FLAGS_raft_create_parent_directories) {
            butil::FilePath sub_dir =
                    butil::FilePath(_writer->get_path()).Append(sub_path.DirName());
            rc = _fs->create_directory(sub_dir.value(), &e, true);
        } else {
            rc = create_sub_directory(
                    _writer->get_path(), sub_path.DirName().value(), _fs, &e);
        }
        if (!rc) {
            LOG(ERROR) << "Fail to create directory for " << file_path
                       << " : " << butil::File::ErrorToString(e);
            set_error(file_error_to_os_error(e),
                      "Fail to create directory");
        }
    }
    LocalFileMeta meta;
    _remote_snapshot.get_file_meta(filename, &meta);
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (_cancelled) {
        set_error(ECANCELED, "%s", berror(ECANCELED));
        return;
    }
    scoped_refptr<RemoteFileCopier::Session> session
        = _copier.start_to_copy_to_file(filename, file_path, NULL);
    if (session == NULL) {
        LOG(WARNING) << "Fail to copy " << filename
                     << " path: " << _writer->get_path();
        set_error(-1, "Fail to copy %s", filename.c_str());
        return;
    }
    _cur_session = session.get();
    lck.unlock();
    session->join();
    lck.lock();
    _cur_session = NULL;
    lck.unlock();
    if (!session->status().ok()) {
        set_error(session->status().error_code(), session->status().error_cstr());
        return;
    }
    if (_writer->add_file(filename, &meta) != 0) {
        set_error(EIO, "Fail to add file to writer");
        return;
    }
    if (_writer->sync() != 0) {
        set_error(EIO, "Fail to sync writer");
        return;
    }
}
```

打开快照
---
```cpp
SnapshotReader* LocalSnapshotStorage::open() {
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (_last_snapshot_index != 0) {
        const int64_t last_snapshot_index = _last_snapshot_index;
        ++_ref_map[last_snapshot_index];
        lck.unlock();
        std::string snapshot_path(_path);
        butil::string_appendf(&snapshot_path, "/" BRAFT_SNAPSHOT_PATTERN, last_snapshot_index);
        LocalSnapshotReader* reader = new LocalSnapshotReader(snapshot_path, _addr,
                _fs.get(), _snapshot_throttle.get());
        if (reader->init() != 0) {
            CHECK(!lck.owns_lock());
            unref(last_snapshot_index);
            delete reader;
            return NULL;
        }
        return reader;
    } else {
        errno = ENODATA;
        return NULL;
    }
}
```

转变为正式快照
---

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


阶段三：Follower 加载快照
===

```cpp
void SnapshotExecutor::load_downloading_snapshot(DownloadingSnapshot* ds,
                                                 const SnapshotMeta& meta) {
    _snapshot_storage->close(_cur_copier);  // _cur_copier: LocalSnapshotStorage
    InstallSnapshotDone* install_snapshot_done =
            new InstallSnapshotDone(this, reader);
    int ret = _fsm_caller->on_snapshot_load(install_snapshot_done);
}
```

```cpp
int LocalSnapshotStorage::close(SnapshotCopier* copier) {
    delete copier;
    return 0;
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
    BRAFT_VLOG << "Set snapshot last_included_index="
              << meta->last_included_index()
              << " last_included_term=" <<  meta->last_included_term();
    std::unique_lock<raft_mutex_t> lck(_mutex);
    if (meta->last_included_index() <= _last_snapshot_id.index) {
        return;
    }
    Configuration conf;
    for (int i = 0; i < meta->peers_size(); ++i) {
        conf.add_peer(meta->peers(i));
    }
    Configuration old_conf;
    for (int i = 0; i < meta->old_peers_size(); ++i) {
        old_conf.add_peer(meta->old_peers(i));
    }
    ConfigurationEntry entry;
    entry.id = LogId(meta->last_included_index(), meta->last_included_term());
    entry.conf = conf;
    entry.old_conf = old_conf;
    _config_manager->set_snapshot(entry);
    int64_t term = unsafe_get_term(meta->last_included_index());

    const LogId last_but_one_snapshot_id = _last_snapshot_id;
    _last_snapshot_id.index = meta->last_included_index();
    _last_snapshot_id.term = meta->last_included_term();
    if (_last_snapshot_id > _applied_id) {  // 从 leader 下载 snapshot 可能会出现
        _applied_id = _last_snapshot_id;
    }
    // NOTICE: not to update disk_id here as we are not sure if this node really
    // has these logs on disk storage. Just leave disk_id as it was, which can keep
    // these logs in memory all the time until they are flushed to disk. By this
    // way we can avoid some corner cases which failed to get logs.

    // last_included_index > last_index
    // 快照包含的日志长度比当前节点日志长度大，这种情况只可能发生在从 leader 下载过来的 snapshot
    if (term == 0) {
        // last_included_index is larger than last_index
        // FIXME: what if last_included_index is less than first_index?
        _virtual_first_log_id = _last_snapshot_id;
        truncate_prefix(meta->last_included_index() + 1, lck);
        return;
    } else if (term == meta->last_included_term()) {  // last_index >= last_included_index
        // Truncating log to the index of the last snapshot.
        // We don't truncate log before the latest snapshot immediately since
        // some log around last_snapshot_index is probably needed by some
        // followers
        if (last_but_one_snapshot_id.index > 0) {
            // We have last snapshot index
            _virtual_first_log_id = last_but_one_snapshot_id;
            truncate_prefix(last_but_one_snapshot_id.index + 1, lck);
        }
        return;
    } else {
        // TODO: check the result of reset.
        _virtual_first_log_id = _last_snapshot_id;
        reset(meta->last_included_index() + 1, lck);
        return;
    }
    CHECK(false) << "Cannot reach here";
}
```



阶段四：Leader 处理响应
===

```cpp
void Replicator::_on_install_snapshot_returned(
            ReplicatorId id, brpc::Controller* cntl,
            InstallSnapshotRequest* request,
            InstallSnapshotResponse* response) {
    std::unique_ptr<brpc::Controller> cntl_guard(cntl);
    std::unique_ptr<InstallSnapshotRequest> request_guard(request);
    std::unique_ptr<InstallSnapshotResponse> response_guard(response);
    Replicator *r = NULL;
    bthread_id_t dummy_id = { id };
    bool succ = true;
    if (bthread_id_lock(dummy_id, (void**)&r) != 0) {
        return;
    }
    if (r->_reader) {
        r->_options.snapshot_storage->close(r->_reader);
        r->_reader = NULL;
        if (r->_options.snapshot_throttle) {
            r->_options.snapshot_throttle->finish_one_task(true);
        }
    }
    std::stringstream ss;
    ss << "received InstallSnapshotResponse from "
       << r->_options.group_id << ":" << r->_options.peer_id
       << " last_included_index " << request->meta().last_included_index()
       << " last_included_term " << request->meta().last_included_term();
    do {
        if (cntl->Failed()) {
            ss << " error: " << cntl->ErrorText();
            LOG(INFO) << ss.str();

            LOG_IF(WARNING, (r->_consecutive_error_times++) % 10 == 0)
                            << "Group " << r->_options.group_id
                            << " Fail to install snapshot at peer="
                            << r->_options.peer_id
                            <<", " << cntl->ErrorText();
            succ = false;
            break;
        }
        if (!response->success()) {
            succ = false;
            ss << " fail.";
            LOG(INFO) << ss.str();
            // Let heartbeat do step down
            break;
        }
        // Success
        r->_next_index = request->meta().last_included_index() + 1;
        ss << " success.";
        LOG(INFO) << ss.str();
    } while (0);

    // We don't retry installing the snapshot explicitly.
    // dummy_id is unlock in _send_entries
    if (!succ) {
        return r->_block(butil::gettimeofday_us(), cntl->ErrorCode());
    }
    r->_has_succeeded = true;
    r->_notify_on_caught_up(0, false);
    if (r->_timeout_now_index > 0 && r->_timeout_now_index < r->_min_flying_index()) {
        r->_send_timeout_now(false, false);
    }
    // dummy_id is unlock in _send_entries
    return r->_send_entries();
}
```

其他：安装快照失败
===

增量更新



TODO
===

> 加载的时候是不是不再接受 Leader 的日志? 或者 Leader 不再发送日志
> 保存快照：
>
> 安装快照：*Follower*
>
> 加载快照：
>
> 对于任一节点来说，这 3 类任务是互斥的，当有
>
> 安装快照时，不再复制日志吗？
> nextIndex 怎么更新
> 快照对应的日志会不会重放？ // 应该没有被删除? 应该是被删除了？