# What’s your problem

## 启动失败

### 服务未启动

```shell
[root@hadoop102 bin]# rabbitmqctl start_app
Starting node rabbit@hadoop102 ...
Error: unable to perform an operation on node 'rabbit@hadoop102'. Please see diagnostics information and suggestions below.

Most common reasons for this are:

 * Target node is unreachable (e.g. due to hostname resolution, TCP connection or firewall issues)
 * CLI tool fails to authenticate with the server (e.g. due to CLI tool's Erlang cookie not matching that of the server)
 * Target node is not running

In addition to the diagnostics info below:

 * See the CLI, clustering and networking guides on https://rabbitmq.com/documentation.html to learn more
 * Consult server logs on node rabbit@hadoop102
 * If target node is configured to use long node names, don't forget to use --longnames with CLI tools

DIAGNOSTICS
===========

attempted to contact: [rabbit@hadoop102]

rabbit@hadoop102:
  * connected to epmd (port 4369) on hadoop102
  * epmd reports: node 'rabbit' not running at all
                  no other nodes on hadoop102
  * suggestion: start the node

Current node details:
 * node name: 'rabbitmqcli-14140-rabbit@hadoop102'
 * effective user's home directory: /var/lib/rabbitmq
 * Erlang cookie hash: 37Uhvo8JDwimQt/VBaZY7g==
```

可能原因:节点的rabbitmq服务未启动

解决:

```shell
 rabbitmq-server -detached
 rabbitmqctl start
```



### 端口被占用

```shell
 ERROR: distribution port 25672 in use by another node: rabbit@hadoop103
```

原因:使用ssh 执行 rabbirmq-server 命令

解决:先关闭占用该端口的进程

```shell
service rabbitmq-server stop
```



