# 总结

```
raft {
    term
    leader_id
    vote

    -- 日志相关
    index
    log_term
}

```


## raft 选举流程

* 一开始所有节点为 follower (term = 0, leader_id = 0, index = 0, log_term = 0)
* 一旦某个节点到达设置的选举时间, 就由 followr 变成 candidate (此时将自己的 term + 1) 进行竞选，发送 MSG_VOTE 信息给所有其他节点，发送的 MSG_VOTE 信息主要携带了 (term, index, log_term) 这 3 个重要信息，用于当其他节点收到 MSG_VOTE 信息时判断是否要投同意票还是拒绝票
* 此时收到 MSG_VOTE 信息的节点，开始响应选举：
    * 以下条件满足一个，投同意票: ()
        * MSG_VOTE 中携带的 term 大于本节点 term
        * MSG_VOTE 中携带的 term 等于本节点 term, 并且 MSG_VOTE 携带的 index 大于或等于本节点  index
    * 否则投反对票
* 随着越来越多的节点响应了这次选举 (不管是同意还是反对)，此时这个竞选的节点就可以得知自己是否可以成为 leader:
    * 只要有收到超过一半的节点投的同意票，那么该节点正式成为 leader，一旦成为 leader 后立即向所有节点发送 MSG_APPEND 信息，宣告自己的统治。因为此时有更可能有些节点因为网络原因还没有投票，也有可能一些节点也变成 candidate 正在进行选举，而该节点最早成为了 leader，所以要一统江山，下发 MSG_APPEDN 试图让所有节点变成 followr~ 此时收到 MSG_APPEND 信息的节点就会乖乖变成 followr (term 为 leader 的 term，vote 和 leader_id 为该 leader id, 并停止一切竞选活动，即 election_elapsed 都会清空)
    * 只要有超过一半的节点投的反对票，此时该节点乖乖变回 follower (将自己原先变成 candidate 时加的 term 减去 1)
    * 否则继续等待，最终等待结果有以下 3 个:
        * 收到超过一半同意票，变成 leader
        * 收到超过一半拒绝票，变成 follower, 这种情况又分为以下 2 种情况
            * 当前 term 落后或者 log_index、log_term 落后, 这种情况代表自己已经不具备成为 leader 的条件
            * 而另一种情况则是，存在很多 candidate 在瓜分选票 (candidate 节点肯定是拒绝票的)，这种情况最糟糕的是会导致所有的 candidate 都得不到足够的选票，全部成为 follower 等待下一次选举，而这种情况是会导致永远选举不出来一个 leader，所以 raft 的实现都将各节点的选举时间进行随机化，这个可以把各节点分散开以至于在大多数情况下只有一个服务器在选举，然后它赢得选举并在其他服务器超时之前发送 MSG_APPEND 宣告统治（）
        * 在等待的时间窗口内，已经有其他节点成为 leader 并宣告了统治，自己也只好成为 follower


    总结: term 相当于一个朝代的旗号，要推翻统治必须改变 term,
    term 是逐渐递增的，如朝代一样，各候选人必须打出自己的旗号，
    以告知其他节点它要建立的统治
