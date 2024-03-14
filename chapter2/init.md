
角色

* NodeManager
* Node

* vote timeout 啥作用啊？

```cpp
int add_service(brpc::Server* server,
                const butil::EndPoint& listen_addr) {

}
```


```cpp
int NodeManager::add_service(brpc::Server* server, const butil::EndPoint& listen_address)
    server->AddService(file_service(), brpc::SERVER_DOESNT_OWN_SERVICE);

    server->AddService(new RaftServiceImpl(listen_address), brpc::SERVER_OWNS_SERVICE);

    server->AddService(new RaftStatImpl, brpc::SERVER_OWNS_SERVICE);

    server->AddService(new CliServiceImpl, brpc::SERVER_OWNS_SERVICE);
```



添加 raft 组

```cpp
int NodeImpl::init(const NodeOptions& options)
    ...

    _conf.id = LogId();
    // if have log using conf in log, else using conf in options
    if (_log_manager->last_log_index() > 0) {
        _log_manager->check_and_set_configuration(&_conf);
    } else {
        _conf.conf = _options.initial_conf;
    }

    global_node_manager->add(this);

    init_fsm_caller(LogId(0, 0));

    ...
```