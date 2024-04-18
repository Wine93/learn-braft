选主优化
===

表格：

| 选主算法 | 作用 | 适用场景 | |
| :--- | :--- | :--- | :--- |
| 选举超时随机化 | 避免选举风暴 | 选举时间不确定 | 高并发场景 |
| 检查 qurom | 避免脑裂 | 选举时间不确定 | 高并发场景 |
| leader lease | 避免 leader 还是 leader | 选举时间不确定 | 高并发场景 |

选举超时随机化
---

pre vote
---

no-op
---

no-op 的作用

* https://github.com/baidu/braft/issues?q=is%3Aissue+noop
* https://zhuanlan.zhihu.com/p/362679439
* https://zhuanlan.zhihu.com/p/30706032

幽灵复现

* https://mp.weixin.qq.com/s?__biz=MzIzOTU0NTQ0MA==&mid=2247494453&idx=1&sn=17b8a97fe9490d94e14b6a0583222837&scene=21#wechat_redirect
* https://zhuanlan.zhihu.com/p/652849109

braft log recovery
* https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md#log-recovery


心跳
---

check qurom
---

leader lease
---

* 作用：解决 leader 还是 leader 的问题，防止 stale read
* API 介绍，使用
