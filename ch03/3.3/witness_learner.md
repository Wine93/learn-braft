概览
===

Raft 除了 `Follower`、`Candidate`、`Leader` 这 3 个角色外，还引入了 `Witness` 与 `Learner`：

* Witness：只参与投票不存储数据；在不降低可用性的同时降低成本。
* Learner：不参与投票只复制数据；在不降低性能的情况下提供多余的副本。

Witness
===

`Witness` 具有以下特征：

* 参与选举投票，但不能成为 Leader；有些实现允许其短暂成为 Leader，再主动转移给其他节点
* 参与日志投票，但不存储日志、不安装快照
* 关于 `Witness` 的更多设计参见 [braft 文档](https://github.com/baidu/braft/blob/master/docs/cn/witness.md)

应用场景
---

**三中心五副本**

有些追求可用性的业务会采用三中心五副本的部署方式，相对于双中心来说，其可用性会更强：
* 双中心：一个中心不可用，服务可能就不可用（取决于不可用中心是否为多数节点所在中心）
* 三中心：只有当两个中心同时不可用时，服务才不可用

但是三中心意味着需要更多的资源，这时候可以将一个副本设置成 `Witness`，而部署 `Witness` 的节点可以为低配机器，因为其不存储数据。这样在保证可用性的同时又降低了成本。

![图 3.15  Witness 的应用](image/3.15.png)

具体实现
---

Leader 向其余节点复制日志结构体为 `LogEntry`，其含有 `data` 字段，当发送给 `Witness` 时，该字段将为空，其余字段不变：

```cpp
struct LogEntry : public butil::RefCountedThreadSafe<LogEntry> {
public:
    EntryType type; // log type
    LogId id;
    std::vector<PeerId>* peers; // peers
    std::vector<PeerId>* old_peers; // peers
    butil::IOBuf data;
    ...
};
```


`Replicator` 发送日志时，将调用 `_prepare_entry` 填充日志的 `meta` 和 `data`，`data` 将放在 RPC 请求的 `attach` 中。如果是 `Witness`，则 `data` 为空：

```cpp
void Replicator::_send_entries() {
    ...
    for (int i = 0; i < max_entries_size; ++i) {
        // 为每一条日志生成 LogEntry
        prepare_entry_rc = _prepare_entry(i, &em, &cntl->request_attachment());
        if (prepare_entry_rc != 0) {
            break;
        }
        request->add_entries()->Swap(&em);
    }
    ...
}

// 如果是 Witness，则 data 为空
int Replicator::_prepare_entry(int offset, EntryMeta* em, butil::IOBuf *data) {
    ...
    if (!is_witness() || FLAGS_raft_enable_witness_to_leader) {
        em->set_data_len(entry->data.length());
        data->append(entry->data);
    }
    entry->Release();
    return 0;
}
```

`Witness` 接受到 `AppendEntries` 请求时，和其他 Follower 一样都是调用 `handle_append_entries_request` 来处理请求。整体处理逻辑是一样的，也需要持久化到本地，只不过其收到的日志的 `data` 是空的，只有日志对应的元数据（`index`、`term`、`type`），所以真正写入磁盘的数据是很小的，就是一些元数据：

```cpp
void NodeImpl::handle_append_entries_request(brpc::Controller* cntl,
                                             const AppendEntriesRequest* request,
                                             AppendEntriesResponse* response,
                                             google::protobuf::Closure* done,
                                             bool from_append_entries_cache) {
    std::vector<LogEntry*> entries;

    // Parse request
    butil::IOBuf data_buf;
    data_buf.swap(cntl->request_attachment());
    ...
    for (int i = 0; i < request->entries_size(); i++) {
        index++;
        const EntryMeta& entry = request->entries(i);
        if (entry.type() != ENTRY_TYPE_UNKNOWN) {
            // (1) 生成 LogEntry
            LogEntry* log_entry = new LogEntry();
            log_entry->AddRef();
            log_entry->id.term = entry.term();
            log_entry->id.index = index;
            log_entry->type = (EntryType)entry.type();
            ...
            // (2) LogEntry 的 data 为空
            if (entry.has_data_len()) {
                int len = entry.data_len();
                data_buf.cutn(&log_entry->data, len);
            }
            // (3) 加入 entries
            entries.push_back(log_entry);
        }
    }

    ...
    // (4) 持久化到本地
    _log_manager->append_entries(&entries, c);
    ...
}
```

关于实现更多细节详见 [PR#398](https://github.com/baidu/braft/pull/398)。

Learner
===

Learner 具有以下特征：

* 不参与选举投票
* 不参与日志投票，但会复制日志

使用场景
---

**提供跨区域只读集群**

* 对于一些一致性要求并不是很高的业务，需要提供跨区域读取时，可以使用 `Learner`
* 如果使用 `Follower` 的话，其会参与日志的 `Quorum` 计算，从而大幅增加写入的时延
* 主集群负责写入，异地提供只读服务

> 当然，Learner 也可以支持读到最新的数据；如果 Raft 库支持 ReadIndex 的话，那可以在 Learner 上实现类似 Follower Read 的功能，详见 [ReadIndex Read](/ch03/3.2/optimization.md#xian-xing-yi-zhi-xing-du)。

![图 3.16  Learner 的应用场景](image/3.16.png)

具体实现
---
目前 braft Learner 并未开源，相关讨论可见 [issue#198](https://github.com/baidu/braft/issues/198)。

参考
===

* [Practical Fast Replication](https://zhuanlan.zhihu.com/p/59991142)
* [字节跳动自研强一致在线 KV &表格存储实践 - 上篇](https://cloud.tencent.com/developer/news/654234)
* [对强一致存储系统的思考](https://zhuanlan.zhihu.com/p/664750172)
* [4.2 两中心同步复制方案（三副本）](https://book.tidb.io/session4/chapter4/two-dc-raft.html)
* [分布式数据库，基于Paxos多副本的两地三中心架构](https://zhuanlan.zhihu.com/p/664750172)