简介
===

超时时间对保证 *Raft* 算法的正确性有着重要的作用，同时也影响着集群的可用性与性能。

```cpp
broadcastTime ≪ electionTimeout ≪ MTBF
```

```cpp
struct NodeOptions {
    // A follower would become a candidate if it doesn't receive any message
    // from the leader in |election_timeout_ms| milliseconds
    // Default: 1000 (1s)
    int election_timeout_ms; //follower to candidate timeout

    // wait new peer to catchup log in |catchup_timeout_ms| milliseconds
    // if set to 0, it will same as election_timeout_ms
    // Default: 0
    int catchup_timeout_ms;

    // Max clock drift time. It will be used to keep the safety of leader lease.
    // Default: 1000 (1s)
    int max_clock_drift_ms;

    // A snapshot saving would be triggered every |snapshot_interval_s| seconds
    // if this was reset as a positive number
    // If |snapshot_interval_s| <= 0, the time based snapshot would be disabled.
    //
    // Default: 3600 (1 hour)
    int snapshot_interval_s;

    // We will regard a adding peer as caught up if the margin between the
    // last_log_index of this peer and the last_log_index of leader is less than
    // |catchup_margin|
    //
    // Default: 1000
    int catchup_margin;

    // If node is starting from an empty environment (both LogStorage and
    // SnapshotStorage are empty), it would use |initial_conf| as the
    // configuration of the group, otherwise it would load configuration from
    // the existing environment.
    //
    // Default: A empty group
    Configuration initial_conf;

    // Run the user callbacks and user closures in pthread rather than bthread
    //
    // Default: false
    bool usercode_in_pthread;

    // The specific StateMachine implemented your business logic, which must be
    // a valid instance.
    StateMachine* fsm;

    // If |node_owns_fsm| is true. |fms| would be destroyed when the backing
    // Node is no longer referenced.
    //
    // Default: false
    bool node_owns_fsm;

    // The specific LogStorage implemented at the business layer, which should be a valid
    // instance, otherwise use SegmentLogStorage by default.
    //
    // Default: null
    LogStorage* log_storage;

    // If |node_owns_log_storage| is true. |log_storage| would be destroyed when
    // the backing Node is no longer referenced.
    //
    // Default: true
    bool node_owns_log_storage;

    // Describe a specific LogStorage in format ${type}://${parameters}
    // It's valid iff |log_storage| is null
    std::string log_uri;

    // Describe a specific RaftMetaStorage in format ${type}://${parameters}
    // Three types are provided up till now:
    // 1. type=local
    //     FileBasedSingleMetaStorage(old name is LocalRaftMetaStorage) will be
    //     used, which is based on protobuf file and manages stable meta of
    //     only one Node
    //     typical format: local://${node_path}
    // 2. type=local-merged
    //     KVBasedMergedMetaStorage will be used, whose under layer is based
    //     on KV storage and manages a batch of Nodes one the same disk. It's
    //     designed to solve performance problems caused by lots of small
    //     synchronous IO during leader electing, when there are huge number of
    //     Nodes in Multi-raft situation.
    //     typical format: local-merged://${disk_path}
    // 3. type=local-mixed
    //     MixedMetaStorage will be used, which will double write the above
    //     two types of meta storages when upgrade an downgrade.
    //     typical format:
    //     local-mixed://merged_path=${disk_path}&&single_path=${node_path}
    //
    // Upgrade and Downgrade steps:
    //     upgrade from Single to Merged: local -> mixed -> merged
    //     downgrade from Merged to Single: merged -> mixed -> local
    std::string raft_meta_uri;

    // Describe a specific SnapshotStorage in format ${type}://${parameters}
    std::string snapshot_uri;

    // If enable, we will filter duplicate files before copy remote snapshot,
    // to avoid useless transmission. Two files in local and remote are duplicate,
    // only if they has the same filename and the same checksum (stored in file meta).
    // Default: false
    bool filter_before_copy_remote;

    // If non-null, we will pass this snapshot_file_system_adaptor to SnapshotStorage
    // Default: NULL
    scoped_refptr<FileSystemAdaptor>* snapshot_file_system_adaptor;

    // If non-null, we will pass this snapshot_throttle to SnapshotExecutor
    // Default: NULL
    scoped_refptr<SnapshotThrottle>* snapshot_throttle;

    // If true, RPCs through raft_cli will be denied.
    // Default: false
    bool disable_cli;

    // If true, this node is a witness.
    // 1. FLAGS_raft_enable_witness_to_leader = false
    //     It will never be elected as leader. So we don't need to init _vote_timer and _election_timer.
    // 2. FLAGS_raft_enable_witness_to_leader = true
    //     It can be electd as leader, but should transfer leader to normal replica as soon as possible.
    //
    // Warning:
    // 1. FLAGS_raft_enable_witness_to_leader = false
    //     When leader down and witness had newer log entry, it may cause leader election fail.
    // 2. FLAGS_raft_enable_witness_to_leader = true
    //     When leader shutdown and witness was elected as leader, if follower delay over one snapshot,
    //     it may cause data lost because witness had truncated log entry before snapshot.
    // Default: false
    bool witness = false;
    // Construct a default instance
    NodeOptions();

    int get_catchup_timeout_ms();
};
```

| 配置项                | 说明 | 默认值（毫秒） |
|:----------------------|:-----|:---------------|
| `election_timeout_ms` |      | 1000           |
| `max_clock_drift_ms`  |      | 1000           |

```cpp
CHECK_EQ(0, _election_timer.init(this, options.election_timeout_ms));
CHECK_EQ(0, _vote_timer.init(this, options.election_timeout_ms + options.max_clock_drift_ms));
CHECK_EQ(0, _stepdown_timer.init(this, options.election_timeout_ms));
CHECK_EQ(0, _snapshot_timer.init(this, options.snapshot_interval_s * 1000));
```

```cpp
 if (FLAGS_raft_election_heartbeat_factor <= 0){
        LOG(WARNING) << "raft_election_heartbeat_factor flag must be greater than 1"
                     << ", but get "<< FLAGS_raft_election_heartbeat_factor
                     << ", it will be set to default value 10.";
        FLAGS_raft_election_heartbeat_factor = 10;
    }
    return std::max(election_timeout / FLAGS_raft_election_heartbeat_factor, 10);
```

timer 超时时间
===

| timer            | 说明           | 默认超时时间（毫秒） |
|:-----------------|:---------------|:---------------------|
| `ElectionTimer`  | 选举超时       | [1000,2000]          |
| `VoteTimer`      | 投票超时       | [2000,3000]          |
| `StepdownTimer`  | 检查 Leader 是否还为 Leader               |  1000                    |
| `SnapshotTimer`  |                |   3600000                   |
| `HeartbeatTimer` | 心跳间隔时间， |        100              |

rpc 请求超时时间
===

| 请求                 | 说明 | 默认超时时间 | 用户自定义 |
|:---------------------|:-----|:-------------|:-----------|
| 心跳请求             |      |              |            |
| `PreVoteRequest`     |      |              |            |
| `RequestVoteRequest` |      |              |            |
| `AppendEntries`      |      |              |            |
| `InstallSnapshot`    |      |              |            |
