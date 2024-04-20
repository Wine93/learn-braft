
角色

* NodeManager
* Node

* vote timeout 啥作用啊？

目录
===

* [简介](#简介)
* [整体流程](#整体流程)
* [具体实现](#具体实现)

整体流程
---

*Raft* 初始化流程主要为以下 6 个步骤：
* (1)
* (2)

具体实现
---

### 步骤二：增加 *Raft* 相关的 *Service* 至 *BRPC Server*

`braft::add_service` 会增加以下 4 个 *Service*：
* `FileService`：用于 *Follower* 安装快照时，向 *Leader* 下载对应文件
* `RaftService`：核心服务，处理 *Raft* 的核心逻辑
* `RaftStat`: 客观性的一部分
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

### 步骤三：初始化 *Raft Node*，并加入 *Raft Group*

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
int NodeImpl::init(const NodeOptions& options)
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

    // RaftMetaStorage, 用来存放一些RAFT算法自身的状态数据， 比如term, vote_for等信息.
    // LogStorage, 用来存放用户提交的WAL
    // SnapshotStorage, 用来存放用户的Snapshot以及元信息.

    _conf.id = LogId();
    // if have log using conf in log, else using conf in options
    if (_log_manager->last_log_index() > 0) {
        _log_manager->check_and_set_configuration(&_conf);
    } else {
        _conf.conf = _options.initial_conf;
    }

    if (!global_node_manager->add(this)) {
        ...
        return -1;
    }

    init_fsm_caller(LogId(0, 0));

    ...
```

```cpp
typedef std::string GroupId;  // 表示复制组的 ID
typedef std::multimap<GroupId, NodeImpl* > GroupMap;
```