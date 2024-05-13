流程详解
===

前置流程：用户创建 BRPC Server，并调用 `braft::add_service` 将 Raft 相关的 Service 加入到 BRPC Server 中，并启动 BRPC Server

1. 用户创建 `braft::node`，并调用 `node->init` 接口
2. 启动任务队列 `ApplyTaskQueue`
3. 遍历一遍日志，读取每条日志的 `Header` (24 字节)：
    * 2.1 读取最新的配置日志
    * 2.2 构建索引（`LogIndex` 到文件 `offset`），便于读取时快速定位
    * 2.3 获得日志的 `FirstIndex` 与 `LastIndex`
4. 回调用户状态机的 `on_snapshot_load` 来加载快照，等待快照加载完成
5. 从日志（包含快照）或用户指定配置初始化集群列表
6. 加载 Raft Meta，即 `currentTerm` 与 `votedFor`
7. 启动快照定时器
8. 将自身角色变为 `Follower`，并启动选举定时器
9. 将节点加入 Raft Group
10. 至此，初始化完成，节点将等待选举超时后发起选举

从以上可以

流程概览
---

* 2：日志并未回放，因为不知道节点的 `CommitIndex`；
* 2.3: 作用是啥？

ApplyTaskQueue
---

这是一个串行执行的任务队列，所有回调给用户状态机的任务都需进入该队列，

* `SNAPSHOT_SAVE`

| 任务类型        | 说明                   |                                 |
|:----------------|:-----------------------|:--------------------------------|
| COMMITTED       | 日志被提交             | 若日志类型为配置文件，则回调 `` |
| SNAPSHOT_SAVE   | 创建快照               |                                 |
| SNAPSHOT_LOAD   | 加载快照               |                                 |
| LEADER_STOP     | Leader 转换为 Follower | on_leader_start                 |
| LEADER_START    |                        |                                 |
| START_FOLLOWING |                        |                                 |
| STOP_FOLLOWING  |                        |                                 |
| ERROR           |                        |                                 |

持久化存储
---

Raft 拥有以下 3 个持久化存储，这些都需要在节点重启时进行重建：

* RaftMetaStorage: 保存 Raft 算法自身的状态信息，即 `(Term, votedFor)` 这个二元组
* LogStorage: 存储用户日志以及元数据
* SnapshotStorage: 存储用户快照以及元数据

Raft 元数据：
```proto
message StablePBMeta {
    required int64 term = 1;
    required string votedfor = 2;
};
```

Log 元数据：
```proto
message LogPBMeta {
    required int64 first_log_index = 1;
};
```

Snapshot 元数据：
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

具体实现
===

增加 Raft Service
---

`braft::add_service` 会增加以下 4 个 *Service*：
* `FileService`：用于 *Follower* 安装快照时，向 *Leader* 下载对应文件
* `RaftService`：核心服务，处理 *Raft* 的核心逻辑
* `RaftStat`: 可观测行的一部分
* `CliService`：控制节点，如配置变更、重置节点列表、转移 *Leader*

```cpp
namespace braft {

// 全局共享的 NodeManager
#define global_node_manager NodeManager::GetInstance()

int add_service(brpc::Server* server,
                const butil::EndPoint& listen_addr) {
    global_init_once_or_die();
    return global_node_manager->add_service(server, listen_addr);
}

}  // namespace braft
```

```cpp
int NodeManager::add_service(brpc::Server* server, const butil::EndPoint& listen_address) {
    ...

    // FileService
    server->AddService(file_service(), brpc::SERVER_DOESNT_OWN_SERVICE);

    // RaftService
    server->AddService(new RaftServiceImpl(listen_address), brpc::SERVER_OWNS_SERVICE);

    // RaftStat
    server->AddService(new RaftStatImpl, brpc::SERVER_OWNS_SERVICE);

    // CliService
    server->AddService(new CliServiceImpl, brpc::SERVER_OWNS_SERVICE);

    ...
}
```

node::init()
---

`NodeImpl::init` 主要完成以下几项工作（详情见以下代码注释）：
* (1) 初始化各类
* (2)
* (3) 初始化节点配置：优先从日志或快照中获取，若其为空则使用用户配置的集群列表 `initial_conf`
* (8) 将节点状态置为 *Follower*
* (9) 启动快照定时器 `_snapshot_timer`
* (10) 如果集群列表**不为空**，则调用 `step_down` 启动选举定时器 `_election_timer`
* (11) 将当前节点加入打对应的复制组中
* (12) 若配置中的集群列表只有当前节点一个，则直接跳过定时器，直接进行选举

归纳来说，主要做以下几类工作：

```cpp
int NodeImpl::init(const NodeOptions& options) {
    ...
    // (1) 初始化以下 4 个定时器
    //    `_election_timer`：选举定时器
    //    `_vote_timer`：投票定时器
    //    `_stepdown_timer`：降级定时器
    //    `_snapshot_timer`：快照定时器
    CHECK_EQ(0, _election_timer.init(this, options.election_timeout_ms));
    CHECK_EQ(0, _vote_timer.init(this, options.election_timeout_ms + options.max_clock_drift_ms));
    CHECK_EQ(0, _stepdown_timer.init(this, options.election_timeout_ms));
    CHECK_EQ(0, _snapshot_timer.init(this, options.snapshot_interval_s * 1000));

    // (2) 启动任务执行对垒
    if (bthread::execution_queue_start(&_apply_queue_id, NULL,
                                       execute_applying_tasks, this) != 0) {
        ...
        return -1;
    }

    _apply_queue = execution_queue_address(_apply_queue_id);


    // (3)
    // Create _fsm_caller first as log_manager needs it to report error
    _fsm_caller = new FSMCaller();

    // (4) 初始化
    _leader_lease.init(options.election_timeout_ms);
    _follower_lease.init(options.election_timeout_ms, options.max_clock_drift_ms);

    // (5)
    // log storage and log manager init
    if (init_log_storage() != 0) {
        ...
        return -1;
    }

    // (6) 初始化
    if (init_fsm_caller(LogId(0, 0)) != 0) {
        ...
        return -1;
    }

    // (7)
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

    // (8)
    // snapshot storage init and load
    // NOTE: snapshot maybe discard entries when snapshot saved but not discard entries.
    //      init log storage before snapshot storage, snapshot storage will update configration
    if (init_snapshot_storage() != 0) {
        LOG(ERROR) << "node " << _group_id << ":" << _server_id
                   << " init_snapshot_storage failed";
        return -1;
    }

    // (8)
    butil::Status st = _log_manager->check_consistency();
    if (!st.ok()) {
        ...
        return -1;
    }

    // (9)
    _conf.id = LogId();
    // if have log using conf in log, else using conf in options
    if (_log_manager->last_log_index() > 0) {
        _log_manager->check_and_set_configuration(&_conf);
    } else {
        _conf.conf = _options.initial_conf;
    }

    // (10)
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

    // (11)
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