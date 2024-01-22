# main

> **main 主流程**

```
main
 newRaftNode
   rc := &raftNode{}
   rc.startRaft()
     rc.replayWAL()
       rc.raftStorage := raft.NewMemoryStorage()
       rc.raftStorage.SetHardState(st)
     rc.node = raft.StartNode(c *Config, peers []Peer)
       r := newRaft(c) // *raft
         r.raftlog := newLogWithSize(c.Storage, c.Logger, c.MaxCommittedSizePerReady)
       r.raftlog.append(e) // e := pb.Entry{Type: pb.EntryConfChange, ...}
       r.addNode(peer.ID)
       n := newNode() // node
       go n.run(r)
     rc.serveChannels()
```
