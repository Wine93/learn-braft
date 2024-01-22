# raftlog

---

## roadmap

* [ ] 一条日志经历未持久化、持久化、commit、apply、快照这几个阶段 (跟踪之)
    * [etcd-raft snapshot实现分析](https://zhuanlan.zhihu.com/p/29865583)

---

## 一些备注

#### snapshot 生命周期:
    1. 通过网络传递获取 snapshot
    2. 通过调用 restore 将 snapshot 存储在 unstable_log.snaphot 中
    3. 通过 ready 将 snapshot 递交给应用层处理
    4. 应用层处理结束后调用 stableSnapTo 将 snapshot 从 unstable_log 中清空

    (1) 首先存在于 unstable (此时 unstable 的 entries 为空)
    (2) 接下去持久化，snapshot 会存在于 storage 的 snapshot 中
        (并调用 ApplySnapshot 初始化 storage.entires，
        保存 first_index、first_term), 之后删除 unstable 中的 snapshot
    (3) 所以 log 中的 first_index 取决于 snapshot 存在于 unstable 而是 storage 中

#### raft log:
    log 中的 index 是从 1 开始计数

#### unstable_log:
    offset: 表示第一条 entries 在 raft log 中的 index
            初始化 (newLogWithSize) 时会对其进行设置，初始化为 1
    entries 中的 index: [ offset、offset + 1 ... ]
    snapshot: 其与 entries 只能存在一个

#### storage:
    entries = [ offset + 1, offset + 2 ...]

    entries.offset: 存在 entries[0] 数组中，
            代表的是最后一个被快照的 raft log 的 index,
            所以你能从 entries 中拿到

    first_index(): 返回 entries 中第一条 entry 在 raft log 中的 index
    last_index(): 返回最后一条 entry 在 raft log 中的 index

#### log:
    first_index: entries 第一个 entry 在 raft log 中的 index
    last_index: entries 最后一个 entry 在 raft log 中的 index，如果 entry 为空，返回被快照后的最后一个 index

---

> **struct**

```
pb.Snapshot
    Data      []byte
    Metadata  SnapshotMetadata
        Index      uint64
        Term       uint64
        ConfState  ConfState
            Nodes     []uint64
            Learners  []uint64

pb.hardstate
    Term    uint64
    Vote    uint64
    Commit  uint64
```

```
// 使用在内存中的数组来实现 Storage 接口的结构体
type MemoryStorage struct {
	sync.Mutex

	hardState pb.HardState
	snapshot  pb.Snapshot

	// ents[i] has raft log position i+snapshot.Metadata.Index
	ents []pb.Entry
}
```

```
type unstable struct {
	// 保存还没有持久化的快照数据
	snapshot *pb.Snapshot

	// 还未持久化的数据
	entries []pb.Entry

	// offset 用于保存 entries 数组中的数据的起始 index
	offset  uint64

	logger Logger
}
```

```
type raftLog struct {
	// 用于保存自从最后一次 snapshot 之后提交的数据
	storage Storage

	// 用于保存还没有持久化的数据和快照，这些数据最终都会保存到 storage 中
	unstable unstable

	// committed 数据索引
	committed uint64

	// committed: 保存是写入持久化存储中的最高 index，
    // applied: 保存的是传入状态机中的最高 index
	// 即一条日志首先要提交成功（即 committed), 才能被 applied 到状态机中
	// 因此以下不等式一直成立：applied <= committed
	applied uint64

	logger Logger

	// maxNextEntsSize is the maximum number aggregate byte size of the messages
	// returned from calls to nextEnts.
	maxNextEntsSize uint64
}
```

## 存储结构

![数据排列](https://www.codedump.info/media/imgs/20180922-etcd-raft/raftlog.png)


## storage.go:
```go
func NewMemoryStorage() *MemoryStorage {
	return &MemoryStorage{
		ents: make([]pb.Entry, 1),
	}
}


// 初始返回 0
func (ms *MemoryStorage) LastIndex() (uint64, error) {
	ms.Lock()
	defer ms.Unlock()
	return ms.lastIndex(), nil
}

func (ms *MemoryStorage) lastIndex() uint64 {
	return ms.ents[0].Index + uint64(len(ms.ents)) - 1 // 0 + 1 -1 = 0
}


// 初始返回 1
func (ms *MemoryStorage) FirstIndex() (uint64, error) {
	ms.Lock()
	defer ms.Unlock()
	return ms.firstIndex(), nil
}

func (ms *MemoryStorage) firstIndex() uint64 {
	return ms.ents[0].Index + 1 // 0 + 1
}
```


## raflog 相关调用

```go
rc.raftStorage = raft.NewMemoryStorage()
rc.raftStorage.SetHardState(st)  // 一个空的 HardState

func newLogWithSize(storage Storage, logger Logger, maxNextEntsSize uint64) *raftLog {
	if storage == nil {
		log.Panic("storage must not be nil")
	}
	log := &raftLog{
		storage:         storage,
		logger:          logger,
		maxNextEntsSize: maxNextEntsSize,
	}
	firstIndex, err := storage.FirstIndex() // 返回 1
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	lastIndex, err := storage.LastIndex() // 返回 0
	if err != nil {
		panic(err) // TODO(bdarnell)
	}

	// offset 从持久化之后的最后一个 index 的下一个开始
	log.unstable.offset = lastIndex + 1 // 1
	log.unstable.logger = logger

	// committed 和 applied 从持久化的第一个 index 的前一个开始
    // committed 和 applied 指向的 index 都是包含关系
	log.committed = firstIndex - 1  // 0
	log.applied = firstIndex - 1    // 0

	return log
}


func (l *raftLog) append(ents ...pb.Entry) uint64 {
	// 没有数据，直接返回最后一条日志索引
	if len(ents) == 0 {
		return l.lastIndex()
	}

	// 如果索引小于 committed，则说明该数据是非法的
	if after := ents[0].Index - 1; after < l.committed {
		l.logger.Panicf("after(%d) is out of range [committed(%d)]", after, l.committed)
	}

	// 放入 unstable 存储中
	l.unstable.truncateAndAppend(ents)
	return l.lastIndex()
}


func (u *unstable) truncateAndAppend(ents []pb.Entry) {
	after := ents[0].Index
	switch {
	case after == u.offset+uint64(len(u.entries)):
		// after is the next index in the u.entries
		// directly append
		u.entries = append(u.entries, ents...)
	case after <= u.offset:
		u.logger.Infof("replace the unstable entries from index %d", after)
		// The log is being truncated to before our current offset
		// portion, so set the offset and replace the entries
		u.offset = after
		u.entries = ents
	default:
		// truncate to after and copy to u.entries
		// then append
		u.logger.Infof("truncate the unstable entries before index %d", after)
		u.entries = append([]pb.Entry{}, u.slice(u.offset, after)...)
		u.entries = append(u.entries, ents...)
	}
}

r.raftLog.committed = r.raftLog.lastIndex()
```
