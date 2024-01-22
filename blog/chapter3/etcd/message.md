# message


| 类型                | 含义                                                  | term |
| ------------------- | ----------------------------------------------------- | ---- |
| MSG_HUP             | 开始选举了, tick 直接调用 (step) 处理                 |      |
| MSG_BEAT            | 心跳包(不包含任何日志内容)，leader 用来维护自己的统治 |      |
| MSG_PROPOSE         |                                                       |      |
| MSG_APPEND          |                                                       |      |
| MSG_APPEND_RESP     |                                                       |      |
| MSG_VOTE            |                                                       |      |
| MSG_VOTE_RESP       |                                                       |      |
| MSG_SNAPSHOT        |                                                       |      |
| MSG_HEARTBEAT       |                                                       |      |
| MSG_HEARTBEAT_RESP  |                                                       |      |
| MSG_UNREACHABLE     |                                                       |      |
| MSG_SNAPSHOT_STATUS |                                                       |      |
| MSG_CHECK_QUORUM    |                                                       |      |
| MSG_TRANSFER_LEADER |                                                       |      |
| MSG_TIMEOUT_NOW     |                                                       |      |
| MSG_READ_INDEX      |                                                       |      |
| MSG_READ_INDEX_RESP |                                                       |      |
| MSG_PRE_VOTE        |                                                       |      |
| MSG_PRE_VOTE_RESP   |                                                       |      |
