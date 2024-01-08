
角色

* NodeManager
* Node


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

    global_node_manager->add(this);

    init_fsm_caller(LogId(0, 0));

    ...
```