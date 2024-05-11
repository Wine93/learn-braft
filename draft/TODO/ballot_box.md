选票箱
===

2 个作用：
* 正常日志
* 配置变更的日志

```cpp
// Called by leader, otherwise the behavior is undefined
// Set logs in [first_log_index, last_log_index] are stable at |peer|.
int BallotBox::commit_at(
        int64_t first_log_index, int64_t last_log_index, const PeerId& peer) {

}
```