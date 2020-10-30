# Minor task for Pingcap Internship Interview

基于 `tiup` 进行开发。

一些早期测试功能用的代码存放在[ofey404/tiup-error-inj](https://github.com/ofey404/tiup-error-inj)，还有开发早期的[日志](https://github.com/ofey404/tiup-error-inj/blob/main/develop_log.md)。

## Task Description
> 小作业：实现一个工具，支持在本地一键启动一个 “支持动态错误注入的 TiDB 集群” 用于测试，同时希望工具比较易用。
> 
> 工具可以接受的输入参数：
> - pd/tidb/tikv 二进制文件路径
> - pd/tidb/tikv 实例数
> - 支持以下错误注入的功能
>   - 重启一个 tikv 实例
>   - 给一个 tikv 实例制造网络分区（比如：让一个 tikv 实例连接不上其它 tikv 实例的 advertise-addr）
> 
> 举个例子，你可以实现一个命令行工具
> ```
> 启动集群：cli start cluster --tikv.path bin/tikv-server --tikv.count 3
>
> 重启 tikv-0 实例：cli restart tikv-0
>
> 隔离 tikv-0 实例：cli partition tikv-0
> ```
> 
> 一些参考资料：
> - 一键启动集群可以参考或基于 tiup playground https://github.com/pingcap/tiup/tree/master/components/playground
> - 模拟网络分区可以参考或基于 https://github.com/etcd-io/etcd/blob/master/pkg/proxy
> - tikv advertise-addr 的概念可以考虑 https://docs.pingcap.com/zh/tidb/dev/command-line-flags-for-tikv-configuration#--advertise-addr

## Usage
启动一个支持错误注入的 `playground`：
```bash
# In repository root:
go build
./tiup playground --error-injection
```

原本 `playground` 的所有参数都适用：
```bash
./tiup playground v3.0.10 --db 3 --pd 3 --kv 3
```

### Cmd：`restart`

使用 `pid`选择实例，和 `playground` 的其他子命令（ eg：`scale-in` ）的工作方式保持一致。`pid`可以通过`display`子命令得到。支持一次传入多个`pid`。

```bash
./tiup playground restart --pid 1000
./tiup playground partition --pid 1000 1001
```

 `restart`命令执行之后，对应的 `tikv`  实例重启时，日志中会出现几条`can not register addr to pd`的错误，但是查看`dashboard`仍然能确认`tikv`在线。`tikv`启动后，会一直向`PD`发送心跳信息，因此可以推断重连成功了。

### Cmd：`partition`

同样使用`pid`选择实例。`pid`可以通过`display`子命令得到。支持一次传入多个`pid`。

```bash
./tiup playground partition --pid 1000
./tiup playground partition --pid 1000 1001
```

`partition`命令将会阻断所有向该实例`--advertise-addr`发起的 tcp 连接。

## Design
使用 Golang 来编写。可以利用现有的模块。

说到隔离某个实例，最先想到的功能肯定是 iptables。隔离某个实例，就创建对应的规则。可以使用 --owner 参数来指定对应的进程，参照 [iptable Tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#OWNERMATCH)。

不过它是内核级功能，普通的 go 程序不能像它一样直接控制经过某个端口的包，也不能抢占端口。

> 让一个 tikv 实例连接不上其它 tikv 实例的 advertise-addr

Proxy server 做到的事情是端口转发+处理。

结合 proxy 的特性和隔离的要求，可以做出以下设计：

```plain text
+-----------------------+                                         +--------------------------+
|                       |                                         |                          |
|            +----------+                                         +---------+                |
|            |          <-----------------------------+           |         |                |
|            |Advertise |                             |           |Advertise|                |
|            |Addr      +-------------+               |           |Addr     |                |
|            |          |    +--------v-------+       |           |         |                |
|            +----------+    |                |       |           +---------+                |
|                       |    | Proxy Server 1 |       |           |                          |
|                       |    |                |       |           |                          |
|  tikv1     +----------+    +--------+-------+       |           +--------+      tikv2      |
|            |          |             |               |           |        |                 |
|            |  Listen  |             |               |           | Source |                 |
|            |  Port    +<------------+               +-------------Port   |                 |
|            |          |                                         |        |                 |
|            +----------+                                         +--------+                 |
|                       |                                         |                          |
|                       |                                         |                          |
|            +----------+                                         |                          |
|            |          |                                         +----------+               |
|            |  Source  |                                         |          |               |
|            |  Port    |                                         |  Listen  |               |
|            |          |                                         |  Port    |               |
|            +----------+                                         |          |               |
|                       |                                         +----------+               |
|                       |                                         |                          |
|                       |                                         |                          |
+-----------------------+                                         +--------------------------+
```

在启动每一个实例的时候，指定一个对应的 advertise-addr，同时启动一个 Proxy server 将 advertise-addr 的流量转发到真实的 Listen Port。

初始 proxy 可以设置成直接转发，执行`partition`等命令时，则修改 proxy server 的设置。

这个设计还可以更精细地控制错误的类型，因为设置的每一个 proxy 相当于完全控制了这条连接的延迟、丢包等信息。

proxy server 程序的性质：
- 需要在集群启动期间保持运行。
- 执行 `partition` 命令时，修改后台的 proxy server 的状态。
- 最好能在集群关闭时自动停止。

参照 playground 的实现：
- 不需要考虑持久化。
- 所有实例在同一个单机上运行，不需要考虑将 proxy 运行在多台机器上（来均衡负载）的情况。

start, restart 等功能和 playground 高度重合，可以考虑为 playground 增加一个子命令/flag 这种实现方式。

### partition 命令
- [x] 如何更改 Proxy 设置？
  - 在执行 partition 命令的时候，用进程间通信方式告知对应的 Proxy Server。服务没有中断时间，但实现较为复杂。
  - 或者直接关闭 Proxy Server 然后用新的配置重启一次。实现简单，但是下线/上线之间服务会中断，不利于频繁/精细的调节网络状况。

发现`playground`自带一个方案，用http请求和后台运行的`playground`实例通信，把这个机制拿来用了。

## Implementation

所有的修改都在[components/playground/](https://github.com/ofey404/tiup-error-inj2/tree/main/components/playground)目录下进行。`playground`本身是一个子模块，可以通过如下的方式调试：

```bash
cd components/playground
go build
tiup --binpath ./playground playground --error--injection
```

`playground`的子命令（eg：`partition`和`--help`）无法单纯通过`--binpath`的方式测试到，会出现解析flag错误的bug。

可以通过如下的方式进行调试：

```bash
cp ./playground ~/.tiup/components/playground/v1.2.1/tiup-playground
tiup playground --help
```

### Flag: `--error-injection`

打开这个flag以后会发生的事：

- [bootOptions](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/main.go#L97)中新增的`err_inj`字段会被置为`true`
- [bootCluster](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L735)的装载实例阶段：
  - 给每一个`TiKVInstance`[分配一个AdvertisePort](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L803)
- [bootCluster](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L735)的启动实例阶段：
  - 给每个`TiKVInstance`[按照AdvertisePort启动一个代理](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L847)
  - Start函数会按照AdvertisePort启动TiKVInstance



对数据结构做的主要改动：

`bootOptions`增加了`err_inj`的[属性](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/main.go#L59)。

`TiKVInstance`增加`AdvertisePort`的[属性](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/instance/tikv.go#L35)，更改了对应的[Start方法](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/instance/tikv.go#L76)。

`playground`增加了[列表`proxys`](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L70)，用于存放错误注入使用的代理服务器。



### Cmd: `restart`

`playground`的命令-后台通信机制是，主进程开一个http服务器，然后监听进来的命令。

执行命令时，经历了这样的事，以`display`命令为例：

- `newDisplay()`设置的[display函数](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/command.go#L232)被调用，向`playground`发送一条请求。
- `playground`调用对应的[handleDisplay函数](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L106)来处理请求，返回数据。



`restart`命令的解析侧和`display`等并没有太大区别，略过。

[服务侧`handleRestart`的实现](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L163)如下：

- `syscall.Kill`掉实例对应的进程，但是保留`TiKVInstance`的数据结构在`playground.tikvs`中。
- 然后重新调用`startInstance`。
  - `startInstance`需要一份`context`，这样在`playground`进程关闭的时候，它创建的所有子进程也会一并关闭。
  - WARNING: 这里做了一个临时的更动，在[`bootCluster`函数中](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L777)保存下了一份`context，然后在`handleRestart`中使用了它。这并不是一个好事，因为并不是这个对象所有的部分都会使用这个context，有的是直接在对象启动的时候通过参数传进去的，这种不一致性可能会在将来的维护中造成bug。



### Cmd: `partition`

同样讨论[服务侧的`handlePartition`](https://github.com/ofey404/tiup-error-inj2/blob/b3ff13573da34b9cdb3ebd7775b3e797ff684740/components/playground/playground.go#L131)。

简单遍历代理列表，根据`pid`查`ip:port`，改变`ip:port`对应的代理的设置。

## Milestone

- 201024 - 201025：全天满课……
- 201025 - 201027：阅读必要的文档，实验了一些功能，详见[开发日志](./develop_log.md)
- 201027 - 201029：阅读 `tiup` 的代码，在其基础上实现了一些功能。
- 201029 - 201030：完成各个命令的开发、测试，编写文档。