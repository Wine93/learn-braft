
参考：https://github.com/baidu/braft/blob/master/docs/cn/server.md
# 为什么操作一定需要幂等

```
apply不一定成功，如果失败的话会设置done中的status，并回调。on_apply中一定是成功committed的，但是apply的结果在leader发生切换的时候存在false negative, 即框架通知这次WAL写失败了， 但最终相同内容的日志被新的leader确认提交并且通知到StateMachine. 这个时候通常客户端会重试(超时一般也是这么处理的), 所以一般需要确保日志所代表的操作是幂等的
```