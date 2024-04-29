简介
===

超时时间对保证 *Raft* 算法的正确性有着重要的作用，同时也影响着集群的可用性与性能。

```cpp
broadcastTime ≪ electionTimeout ≪ MTBF
```

timer 超时时间
===

>

| timer           | 说明                    | 默认超时时间（毫秒） | 用户自定义 |
|:----------------|:------------------------|:---------------------|:-----------|
| election timer  | 选举超时，默认为 1 秒， | [0,1000] + 1000      |            |
| vote timer      | 投票超时，默认为 2 秒， | [1000,2000] + 1000   |            |
| stepdown timer  |                         |                      |            |
| snapshot timer  |                         |                      |            |
| heartbeat timer | 心跳间隔时间，                         |                      |            |

rpc 请求超时时间
===

| 请求                 | 说明 | 默认超时时间 | 用户自定义 |
|:---------------------|:-----|:-------------|:-----------|
| `PreVoteRequest`     |      |              |            |
| `RequestVoteRequest` |      |              |            |
| `AppendEntries`      |      |              |            |
| 心跳请求             |      |              |            |
| `InstallSnapshot` | | | |
