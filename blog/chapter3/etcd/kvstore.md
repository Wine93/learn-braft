# kvstore

# 源码阅读 (4): kvstore

## roadmap

* [ ] set 操作请求到 follower/candidate/leader 节点，它们分别是如何处理的？
* [ ] 跟踪一个 set 操作产生的 log 从广播到 commit，再到 apply 的过程

---

## 应用层

```
func (h *httpKVAPI) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	key := r.RequestURI
	defer r.Body.Close()
	switch {
	case r.Method == "PUT":
		v, err := ioutil.ReadAll(r.Body)
		if err != nil {
			log.Printf("Failed to read on PUT (%v)\n", err)
			http.Error(w, "Failed on PUT", http.StatusBadRequest)
			return
		}

		h.store.Propose(key, string(v))

        ...
     }

     ...
}


func (s *kvstore) Propose(k string, v string) {
	var buf bytes.Buffer
	if err := gob.NewEncoder(&buf).Encode(kv{k, v}); err != nil {
		log.Fatal(err)
	}
	// kvstore 的 proposeC 便是应用初始化时创建的与 raftNode 结构之间进行请求传输的 channel，
	// 对于更新请求，kvstore 将其通过 proposeC 直接提交给了 raftNode
	// (raftnode 在 serveChannels 函数中处理)
	s.proposeC <- buf.String()
}


// kvstore 启动时运行的协程
func (rc *raftNode) serveChannels() {
    ...

	// send proposals over raft
	go func() {
		for rc.proposeC != nil && rc.confChangeC != nil {
			select {
			case prop, ok := <-rc.proposeC:
				if !ok {
					rc.proposeC = nil
				} else {
					rc.node.Propose(context.TODO(), []byte(prop))
				}

             ...
		}
	}()
}

```

## raft 算法层

```
func (n *node) Propose(ctx context.Context, data []byte) error {
	return n.stepWait(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}


func (n *node) stepWait(ctx context.Context, m pb.Message) error {
	return n.stepWithWaitOption(ctx, m, true)
}


func (n *node) stepWithWaitOption(ctx context.Context, m pb.Message, wait bool) error {

    ...

	ch := n.propc
	pm := msgWithResult{m: m}
	if wait {
		pm.result = make(chan error, 1)
	}
	select {
	case ch <- pm:
		if !wait {
			return nil
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}

    // 等待请求结束
	select {
	case err := <-pm.result:
		if err != nil {
			return err
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	return nil
}

```

## 核心处理函数

```
func (n *node) run(r *raft) {
    for {
        ...

        select {
            case pm := <-propc: // 处理本地收到的提交值
			m := pm.m
			m.From = r.id
			err := r.Step(m)
			if pm.result != nil {
				pm.result <- err
				close(pm.result)
			}
        }
    }
}


func (r *raft) Step(m pb.Message) error {
    ...

    for {

        ...

        switch m.Type {

        ...

        default:
		    // 根据角色调用不同的 step 函数：
		    //     (1) leader: stepLeader
		    //     (2) candidate: stepCandidate
		    //     (3) follower: stepFollower
		    err := r.step(r, m)
		    if err != nil {
		    	return err
		    }
        }
    }
}


func stepLeader(r *raft, m pb.Message) error {
    switch m.Type {

    ...

    case pb.MsgProp:
		if len(m.Entries) == 0 { // 不能提交空数据
			r.logger.Panicf("%x stepped empty MsgProp", r.id)
		}
		if r.prs.getProgress(r.id) == nil { // 检查是否在集群中
			return ErrProposalDropped
		}
		if r.leadTransferee != None { // 当前正在转换 leader 过程中，不能提交
			return ErrProposalDropped
		}

		for i := range m.Entries {
			e := &m.Entries[i]
			if e.Type == pb.EntryConfChange {
				if r.pendingConfIndex > r.raftLog.applied {
					m.Entries[i] = pb.Entry{Type: pb.EntryNormal}
				} else {
					r.pendingConfIndex = r.raftLog.lastIndex() + uint64(i) + 1
				}
			}
		}

		// 添加数据到 log 中
		if !r.appendEntry(m.Entries...) {
			return ErrProposalDropped
		}
		// 向集群其他节点广播 append 消息
		r.bcastAppend()
		return nil
    }

    ...
}


// 向所有 follower 发送 append 消息
func (r *raft) bcastAppend() {
	r.prs.visit(func(id uint64, _ *Progress) {
		if id == r.id {
			return
		}

		r.sendAppend(id)
	})
}


// 向 to 节点发送 append 消息
func (r *raft) sendAppend(to uint64) {
	r.maybeSendAppend(to, true)
}


func (r *raft) maybeSendAppend(to uint64, sendIfEmpty bool) bool {
	pr := r.prs.getProgress(to)
	if pr.IsPaused() {
		return false
	}
	m := pb.Message{}
	m.To = to

	// 从该节点的 Next 的上一条数据获取 term
	term, errt := r.raftLog.term(pr.Next - 1)
	// 获取从该节点的 Next 之后的 entries，总和不超过 maxMsgSize
	ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)
	if len(ents) == 0 && !sendIfEmpty {
		return false
	}

	if errt != nil || erre != nil { // send snapshot if we failed to get term or entries
		// 如果前面过程中有错，那说明之前的数据都写到快照里了，尝试发送快照数据过去
		if !pr.RecentActive { // 如果该节点当前不可用
			r.logger.Debugf("ignore sending snapshot to %x since it is not recently active", to)
			return false
		}

		// 尝试发送快照
		m.Type = pb.MsgSnap
		snapshot, err := r.raftLog.snapshot()
		if err != nil {
			if err == ErrSnapshotTemporarilyUnavailable {
				r.logger.Debugf("%x failed to send snapshot to %x because snapshot is temporarily unavailable", r.id, to)
				return false
			}
			panic(err) // TODO(bdarnell)
		}
		if IsEmptySnap(snapshot) { // 不能发送空快照
			panic("need non-empty snapshot")
		}
		m.Snapshot = snapshot
		sindex, sterm := snapshot.Metadata.Index, snapshot.Metadata.Term
		r.logger.Debugf("%x [firstindex: %d, commit: %d] sent snapshot[index: %d, term: %d] to %x [%s]",
			r.id, r.raftLog.firstIndex(), r.raftLog.committed, sindex, sterm, to, pr)
		pr.becomeSnapshot(sindex) // 该节点进入接收快照的状态
		r.logger.Debugf("%x paused sending replication messages to %x [%s]", r.id, to, pr)
	} else { // 否则就是简单的发送 append 消息
		m.Type = pb.MsgApp
		m.Index = pr.Next - 1
		m.LogTerm = term
		m.Entries = ents
		// append 消息需要告知当前 leader 的 commit 索引
		m.Commit = r.raftLog.committed
		if n := len(m.Entries); n != 0 { // 如果发送过去的 entries 不为空
			switch pr.State {
			// optimistically increase the next when in ProgressStateReplicate

			// 如果该节点在接受副本的状态
			// 得到待发送数据的最后一条索引
			case ProgressStateReplicate:
				last := m.Entries[n-1].Index
				// 直接使用该索引更新 Next 索引
				pr.optimisticUpdate(last)
				pr.ins.add(last)
			case ProgressStateProbe: // 在 probe 状态时，每次只能发送一条 app 消息
				pr.pause()
			default:
				r.logger.Panicf("%x is sending append in unhandled state %s", r.id, pr.State)
			}
		}
	}
	r.send(m)
	return true
}
```
