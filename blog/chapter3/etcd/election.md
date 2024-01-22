# 一次选举流程


## 发起投票请求

```
r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
```

```
func (rc *raftNode) serveChannels() {
    ...

    ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

    for {
		select {
		case <-ticker.C:
			rc.node.Tick()
        }

        ...
    }
}


func (n *node) Tick() {
	select {
	case n.tickc <- struct{}{}: // 向 tick channel 写入空数据，唤醒之
	case <-n.done:
	default:
        ...
	}
}


// 每个 raft 节点都拥有的处理协程
func (n *node) run(r *raft) {
    ...

    for {
        ...

        select {
        // 定期器，根据节点类型的不同，定时触发不同的调用函数:
		//     (1) followr: tickElection
		case <-n.tickc:
			r.tick()
        }

        ...
    }
}


// follower 以及 candidate 的 tick 函数，在 r.electionTimeout 之后被调用
func (r *raft) tickElection() {
	r.electionElapsed++

	// 如果可以被提升为 leader，同时选举时间也到了
	if r.promotable() && r.pastElectionTimeout() {
		r.electionElapsed = 0
		// 发送 HUP 消息是为了重新开始选举
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup}) // From: 1, Term: 0
	}
}

func (r *raft) Step(m pb.Message) error {
	switch {
	case m.Term == 0: // 来自本地的消息
    ...
    }

    switch m.Type {
	case pb.MsgHup: // 收到 HUP 消息，说明准备进行选举
		if r.state != StateLeader { // 当前不是 leader
			if !r.promotable() {
				return nil
			}
			// 取出 [applied+1,committed+1] 之间的消息，即得到还未进行 applied 的日志列表
			ents, err := r.raftLog.slice(r.raftLog.applied+1, r.raftLog.committed+1, noLimit)
			if err != nil {
				r.logger.Panicf("unexpected error getting unapplied entries (%v)", err)
			}

			// 如果其中有 config 消息，并且 commited > applied，说明当前还有没有 apply 的 config 消息，
			// 这种情况下不能开始投票
			if n := numOfPendingConf(ents); n != 0 && r.raftLog.committed > r.raftLog.applied {
				r.logger.Warningf("%x cannot campaign at term %d since there are still %d pending configuration changes to apply", r.id, r.Term, n)
				return nil
			}

			if r.preVote {
				r.campaign(campaignPreElection)
			} else {
				// 进入竞选状态
				r.campaign(campaignElection)
			}
		} else {
			r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
		}

     ...
   }


// 调用 becomeCandidate 把自己切换到 candidate 模式，并递增 Term 值
// 然后再将自己的 Term 及日志信息发送给其他的节点，请求投票
func (r *raft) campaign(t CampaignType) {
	if !r.promotable() {
		// This path should not be hit (callers are supposed to check), but
		// better safe than sorry.
		r.logger.Warningf("%x is unpromotable; campaign() should have been called", r.id)
	}
	var term uint64
	var voteMsg pb.MessageType
	if t == campaignPreElection {
		r.becomePreCandidate()
		voteMsg = pb.MsgPreVote
		// PreVote RPCs are sent for the next term before we've incremented r.Term.
		term = r.Term + 1
	} else {
		r.becomeCandidate()
		voteMsg = pb.MsgVote
		term = r.Term
	}
	if _, _, res := r.poll(r.id, voteRespMsgType(voteMsg), true); res == electionWon {
		// We won the election after voting for ourselves (which must mean that
		// this is a single-node cluster). Advance to the next state.
		if t == campaignPreElection {
			r.campaign(campaignElection)
		} else {
			r.becomeLeader()
		}
		return
	}
	for id := range r.prs.nodes {
		if id == r.id {
			continue
		}
		r.logger.Infof("%x [logterm: %d, index: %d] sent %s request to %x at term %d",
			r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), voteMsg, id, r.Term)

		var ctx []byte
		if t == campaignTransfer {
			ctx = []byte(t)
		}
		r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
	}
}
```

## 响应投票请求:

```go
// 同意投票
r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
r.electionElapsed = 0
r.Vote = m.From

// 拒绝投票
r.send(pb.Message{To: m.From, Term: r.Term, Type: voteRespMsgType(m.Type), Reject: true})
```

```go
// raft 节点启动时注册的 HTTP/TCP handler, 其他节点接受消息的入口函数
func (rc *raftNode) Process(ctx context.Context, m raftpb.Message) error {
	return rc.node.Step(ctx, m)
}


func (n *node) Step(ctx context.Context, m pb.Message) error {
	// ignore unexpected local messages receiving over network
	if IsLocalMsg(m.Type) {
		// TODO: return an error?
		return nil
	}
	return n.step(ctx, m)
}


func (n *node) step(ctx context.Context, m pb.Message) error {
	return n.stepWithWaitOption(ctx, m, false)
}


func (n *node) stepWithWaitOption(ctx context.Context, m pb.Message, wait bool) error {
	if m.Type != pb.MsgProp { // 选举时的 Type 为 MsgVote
		select {
		case n.recvc <- m: // 通过 recev channel 传递
			return nil
		case <-ctx.Done():
			return ctx.Err()
		case <-n.done:
			return ErrStopped
		}
	}
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


func (n *node) run(r *raft) {
    ...

    for {

        ...

        select {
            ...

        	case m := <-n.recvc: // 处理其他节点发送过来的提交值
			    // filter out response message from unknown From.

			    // 需要确保节点在集群中或者不是应答类消息的情况下才进行处理
			    if pr := r.prs.getProgress(m.From); pr != nil || !IsResponseMsg(m.Type) {
				    r.Step(m)
			    }

            ...
        }
    }

}


// Step 的主要作用是处理不同的消息，所以以后当我们想知道 raft 对某种消息的处理逻辑时，到这里找就对了
// 在函数的最后，有个 default 语句，即所有上面不能处理的消息都落入这里，由一个小写的 step 函数处理
// 这个设计的原因是什么呢？
//
// 其实是因为这里的 raft 也被实现为一个状态机，它的 step 属性是一个函数指针，根据当前节点的不同角色，
// 指向不同的消息处理函数：stepLeader/stepFollower/stepCandidate
// 与它类似的还有一个 tick 函数指针，根据角色的不同，也会在 tickHeartbeat 和 tickElection 之间来回切换，
// 分别用来触发定时心跳和选举检测, 这里的函数指针感觉像实现了OOP里的多态。
func (r *raft) Step(m pb.Message) error {
	switch {
	case m.Term == 0: // 来自本地的消息
	case m.Term > r.Term: // 消息的 Term 大于节点当前的 Term
		if m.Type == pb.MsgVote || m.Type == pb.MsgPreVote { // 如果收到的是投票类消息
			// 当 context 为 campaignTransfer 时表示强制要求进行竞选
			force := bytes.Equal(m.Context, []byte(campaignTransfer))

			// 是否在租约期以内
			inLease := r.checkQuorum && r.lead != None && r.electionElapsed < r.electionTimeout

			// 如果非强制，而且又在租约期以内，就不做任何处理
			// 非强制又在租约期内可以忽略选举消息，见论文的 4.2.3，
			// 这是为了阻止已经离开集群的节点再次发起投票请求
			if !force && inLease {
				return nil
			}
		}
		switch {
		// 注意 Go 的 switch case 不做处理的话是不会默认走到 default 情况的
		case m.Type == pb.MsgPreVote:
		case m.Type == pb.MsgPreVoteResp && !m.Reject: // 在应答一个 prevote 消息时不对任期 term 做修改
		default:
			if m.Type == pb.MsgApp || m.Type == pb.MsgHeartbeat || m.Type == pb.MsgSnap {
				r.becomeFollower(m.Term, m.From)
			} else {
				r.becomeFollower(m.Term, None)
			}
		}

	// 消息的 Term 小于节点自身的 Term，同时消息类型是心跳消息或者是 append 消息
	case m.Term < r.Term:
		if (r.checkQuorum || r.preVote) && (m.Type == pb.MsgHeartbeat || m.Type == pb.MsgApp) {
			// 收到了一个节点发送过来的更小的term消息。
			// 这种情况可能是因为消息的网络延时导致，但是也可能因为该节点由于网络分区导致了它递增了 term 到一个新的任期
			// 这种情况下该节点不能赢得一次选举，也不能使用旧的任期号重新再加入集群中。
			// 如果 checkQurom 为 false，这种情况可以使用递增任期号应答来处理。
			// 但是如果 checkQurom 为 True, 此时收到了一个更小的 term 的节点发出的 HB 或者 APP 消息，于是应答一个 appresp 消息，试图纠正它的状态
			r.send(pb.Message{To: m.From, Type: pb.MsgAppResp})
		} else if m.Type == pb.MsgPreVote {
			r.send(pb.Message{To: m.From, Term: r.Term, Type: pb.MsgPreVoteResp, Reject: true})
		} else { // 除了上面的情况以外，忽略任何 term 小于当前节点所在任期号的消息
		}
		return nil
	}

	switch m.Type {
    ...

	case pb.MsgVote, pb.MsgPreVote: // 收到投票类的消息
		if r.isLearner {
			return nil
		}

		// 其他节点在接受到这个请求后，会首先比较接收到的 Term 是不是比自己的大，
		// 以及接受到的日志信息是不是比自己的要新，从而决定是否投票
		canVote := r.Vote == m.From ||
			(r.Vote == None && r.lead == None) ||
			(m.Type == pb.MsgPreVote && m.Term > r.Term)
		if canVote && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
			r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
			if m.Type == pb.MsgVote {
				// 保存下来给哪个节点投票了
				r.electionElapsed = 0
				r.Vote = m.From
			}
		} else { // 否则拒绝投票
			r.send(pb.Message{To: m.From, Term: r.Term, Type: voteRespMsgType(m.Type), Reject: true})
		}

    ...
	}
	return nil
}
```

## 处理投票处理响应

```
前面过程忽略，参考响应投票请求（以上）

func (r *raft) Step(m pb.Message) error {
    ...

    switch m.Type {
    	default:
		// 根据角色调用不同的 step 函数：
		//     (1) follower: stepFollower
		//     (2) leader: stepLeader
		//     (3) candidate: stepCandidate
		err := r.step(r, m)
		if err != nil {
			return err
		}
    }
}


// whether they respond to MsgVoteResp or MsgPreVoteResp.
func stepCandidate(r *raft, m pb.Message) error {
	var myVoteRespType pb.MessageType
	if r.state == StatePreCandidate {
		myVoteRespType = pb.MsgPreVoteResp
	} else {
		myVoteRespType = pb.MsgVoteResp
	}
	switch m.Type {
	case pb.MsgProp:
		r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
		return ErrProposalDropped
	case pb.MsgApp:
		r.becomeFollower(m.Term, m.From) // always m.Term == r.Term
		r.handleAppendEntries(m)
	case pb.MsgHeartbeat:
		r.becomeFollower(m.Term, m.From) // always m.Term == r.Term
		r.handleHeartbeat(m)
	case pb.MsgSnap:
		r.becomeFollower(m.Term, m.From) // always m.Term == r.Term
		r.handleSnapshot(m)

	// 当 candidate 节点收到投票回复后，就会计算收到的选票数目是否大于所有节点数的一半，
	// 如果大于则自己成为 leader，并昭告天下，否则将自己置为 follower
	case myVoteRespType:
		gr, rj, res := r.poll(m.From, m.Type, !m.Reject)
		r.logger.Infof("%x has received %d %s votes and %d vote rejections", r.id, gr, m.Type, rj)
		switch res {
		case electionWon:
			if r.state == StatePreCandidate {
				r.campaign(campaignElection)
			} else {
				r.becomeLeader()
				r.bcastAppend()
			}
		case electionLost:
			// pb.MsgPreVoteResp contains future term of pre-candidate
			// m.Term > r.Term; reuse r.Term
			r.becomeFollower(r.Term, None)
		}
	case pb.MsgTimeoutNow:
		r.logger.Debugf("%x [term %d state %v] ignored MsgTimeoutNow from %x", r.id, r.Term, r.state, m.From)
	}
	return nil
}


// 轮询集群中所有节点，返回一共有多少节点已经进行了投票
func (r *raft) poll(id uint64, t pb.MessageType, v bool) (granted int, rejected int, result electionResult) {
	if v {
		r.logger.Infof("%x received %s from %x at term %d", r.id, t, id, r.Term)
	} else {
		r.logger.Infof("%x received %s rejection from %x at term %d", r.id, t, id, r.Term)
	}
	r.prs.recordVote(id, v)
	return r.prs.tallyVotes()
}


// tallyVotes returns the number of granted and rejected votes, and whether the
// election outcome is known.
func (p *progressTracker) tallyVotes() (granted int, rejected int, result electionResult) {
	for _, v := range p.votes {
		if v {
			granted++
		} else {
			rejected++
		}
	}

	q := p.quorum()

    // 这个结果非常重要，只有大于一半同意或拒绝，最终结果才会确定，否则处于不确定状态
    // 处于不确定状态主要是因为：投票结果是一个一个回复的，有先后顺序
	result = electionIndeterminate

	if granted >= q {
		result = electionWon
	} else if rejected >= q {
		result = electionLost
	}
	return granted, rejected, result
}
```
