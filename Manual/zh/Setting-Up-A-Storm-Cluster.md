# Storm 集群安装配置

本文详细介绍了 Storm 集群的安装配置方法。如果需要在 AWS 上安装 Storm，你应该看一下 [storm-deploy][7] 项目。[storm-deploy][7] 可以自动完成 E2 上 Storm 集群的准备、配置、安装的全部过程，同时还设置好了 Ganglia，方便监控 CPU、磁盘以及网络的使用信息。

如果你在使用 Storm 集群时遇到问题，请先查看“[问题与解决][1]”一文中是否已有相应的解决方案。如果检索不到有效的解决方法，请向社区的邮件列表发送关于问题的邮件。

以下是安装 Storm 的步骤：

1. 安装 ZooKeeper 集群；
2. 在各个机器上安装运行集群所需要的依赖组件；
3. 下载 Storm 安装程序并解压缩到集群的各个机器上；
4. 在 storm.yaml 中添加集群配置信息；
5. 使用 “storm” 脚本启动各机器后台进程。

## 安装 ZooKeeper 集群

Storm 使用 ZooKeeper 来保证集群的一致性。集群中 ZooKeeper 并不是用来进行消息传递的，所以 Storm 对 ZooKeeper 的负载相当低。虽然在大部分场景下单点 ZooKeeper 也勉强够用，但是如果你需要更可靠的 HA 机制或者需要部署大规模 Storm 集群，你最好配置一个 ZooKeeper 集群。ZooKeeper 集群的部署说明请参考[此文][2]。

关于 ZooKeeper 部署的几点说明：

1. ZooKeeper 必须在监控模式下运行。因为 ZooKeeper 是个快速失败系统，如果遇到了故障，ZooKeeper 服务会主动关闭。更多详细信息请参考[此文][3]。
2. 需要设置一个 cron 服务来定时压缩 ZooKeeper 的数据与事务日志。因为 ZooKeeper 的后台进程不会处理这个问题，如果不配置 cron，ZooKeeper 的日志会很快填满磁盘空间。更多详细信息请参考[此文][4]。

## 安装必要的依赖组件

接下来你需要在集群中的所有机器上安装必要的依赖组件，包括：

1. Java 6（推荐使用 JDK 7 以上版本 —— 译者注）
2. Python 2.6.6（推荐使用 Python 2.7.x 版本 —— 译者注）

以上均为在 Storm 上测试通过的版本。Storm 并不保证对其他版本的 Java 或 Python 的支持。

## 下载 Storm 安装程序并解压

接下来就要下载需要的 Storm 发行版，并将 zip 安装文件解压缩到集群中的各个机器上。Storm 的发行版可以在[这里][5]下载（推荐在 Storm 官方网站的[下载页面][6]使用 Apache 的镜像服务下载 —— 译者注）。

## 配置 storm.yaml

Storm 的安装包中包含一个在 conf 目录下的 `storm.yaml` 文件，该文件是用于配置 Storm 集群的各种属性的。你可以在[这里][6]查看各个配置项的默认值。storm.yaml 会覆盖 defaults.yaml 中各个配置项的默认值。以下是几个在安装集群时必须配置的选项：

1) **storm.zookeeper.servers**：这是 Storm 关联的 ZooKeeper 集群的地址列表，此项的配置是如下所示：

```
storm.zookeeper.servers:
  - "111.222.333.444"
  - "555.666.777.888"
```

注意，如果你使用的 ZooKeeper 集群的端口不是默认端口，你还需要相应地配置 **storm.zookeeper.port**。

2) **storm.local.dir**：Nimbus 和 Supervisor 后台进程都需要一个用于存放一些状态数据（比如 jar 包、配置文件等等）的目录。你可以在每个机器上创建好这个目录，赋予相应的读写权限，并将该目录写入配置文件中，如下所示：

```
storm.local.dir: "/mnt/storm"
```

3) **nimbus.host**：集群的工作节点需要知道集群中的哪台机器是主机，以便从主机上下载拓扑以及配置文件，如下所示：

```
nimbus.host: "111.222.333.44"
```

4) **supervisor.slots.ports**：你需要通过此配置项配置每个 Supervisor 机器能够运行的工作进程（worker）数。每个 worker 都需要一个单独的端口来接收消息，这个配置项就定义了 worker 可以使用的端口列表。如果你在这里定义了 5 个端口，那么 Storm 就会在该机器上分配最多 5 个worker。如果定义 3 个端口，那 Storm 至多只会运行三个 worker。此项的默认值是 6700、6701、6702、6703 四个端口，如下所示：

```
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

## 配置外部库与环境变量（可选）

如果你需要使用某些外部库或者定制插件的功能，你可以将相关 jar 包放入 `extlib/` 与 `extlib-daemon` 目录下。注意，`extlib-daemon` 目录仅仅用于存储后台进程（Nimbus，Supervisor，DRPC，UI，Logviewer）所需的 jar 包，例如，HDFS 以及定制的调度库。另外，可以使用`STORM_EXT_CLASSPATH` 和 `STORM_EXT_CLASSPATH_DAEMON` 两个环境变量来配置普通外部库与“仅用于后台进程”外部库的 classpath。

## 使用 “storm” 脚本启动后台进程

最后一步是启动所有的 Storm 后台进程。注意，这些进程必须在严格监控下运行。因为 Storm 是个与 ZooKeeper 相似的快速失败系统，其进程很容易被各种异常错误终止。之所以设计成这种模式，是为了确保 Storm 进程可以在任何时刻安全地停止并且在进程重新启动之后恢复征程。这也是 Storm 不在处理过程中保存任何状态的原因 —— 在这种情况下，如果有 Nimbus 或者 Supervisor 重新启动，运行中的拓扑不会受到任何影响。下面是启动后台进程的方法：

1. **Nimbus**：在 master 机器上，在监控下执行 `bin/storm nimbus` 命令。
2. **Supervisor**：在每个工作节点上，在监控下执行 `bin/storm supervisor` 命令。Supervisor 的后台进程主要负责启动/停止该机器上的 worker 进程。
3. **UI**：在 master 机器上，在监控下执行 `bin/storm ui` 命令启动 Storm UI（Storm UI 是一个可以在浏览器中方便地监控集群与拓扑运行状况的站点）后台进程。可以通过 `http://{nimbus.host}:8080` 来访问 UI 站点。

可以看出，启动后台进程非常简单。同时，各个后台进程也会将日志信息记录到 Storm 安装程序的 logs/ 目录中。


[1]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Troubleshooting.md
[2]: http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html
[3]: http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_supervision
[4]: http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_maintenance
[5]: http://github.com/apache/storm/releases
[6]: https://github.com/apache/storm/blob/master/conf/defaults.yaml
[7]: https://github.com/nathanmarz/storm-deploy/wiki