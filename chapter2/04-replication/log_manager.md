日志的管理
===

* [初始化](#初始化)
* [追加日志](#追加日志)

* 功能
* 性能

* 为什么需要内存缓存？

* 用几句话描述日志管理的作用

* 优化点
    * 内存缓存？// 为什么需要内存缓存？直接线程写入磁盘不行嘛?
    * batch
    * Paprallel append

* brpc execution queue

初始化
---

```cpp
int LogManager::start_disk_thread() {
    ...
    return bthread::execution_queue_start(&_disk_queue,
                                   &queue_options,
                                   disk_thread,
                                   this);
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

        TruncatePrefixClosure* tpc =
                        dynamic_cast<TruncatePrefixClosure*>(done);
                if (tpc) {
                    ret = log_manager->_log_storage->truncate_prefix(
                                    tpc->first_index_kept());
                    break;
                }
    }

    log_manager->set_disk_id(last_id);  // 清理内存
}
```

```cpp
void LeaderStableClosure::Run()
    _ballot_box->commit_at(_first_log_index,
                           _first_log_index + _nentries - 1,
                           _node_id.peer_id);
```

追加日志
---

```cpp
void LogManager::append_entries(
            std::vector<LogEntry*> *entries, StableClosure* done) {

    ...

    if (!entries->empty()) {
        done->_first_log_index = entries->front()->id.index;
        _logs_in_memory.insert(_logs_in_memory.end(), entries->begin(), entries->end());
    }

    done->_entries.swap(*entries);
    int ret = bthread::execution_queue_execute(_disk_queue, done);
    wakeup_all_waiter(lck);
}
```

// 唤醒所有 replicator 进行网络复制？ // 是的
```cpp
void LogManager::wakeup_all_waiter(std::unique_lock<raft_mutex_t>& lck) {
    for (size_t i = 0; i < nwm; ++i) {
        wm[i]->error_code = error_code;
        bthread_t tid;
        bthread_attr_t attr = BTHREAD_ATTR_NORMAL | BTHREAD_NOSIGNAL;
        if (bthread_start_background(
                    &tid, &attr,
                    run_on_new_log, wm[i]) != 0) {
            PLOG(ERROR) << "Fail to start bthread";
            run_on_new_log(wm[i]);
        }
    }
}
```


参考
---

* [从源码吃透共识协议：braft 日志复制](https://zhuanlan.zhihu.com/p/635963776)
* [braft源码分析（二）日志复制、配置变更和快照](https://zhuanlan.zhihu.com/p/169904153)
* [brpc: FlatMap](https://brpc.apache.org/zh/docs/c++-base/flatmap/)
