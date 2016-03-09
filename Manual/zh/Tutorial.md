# 教程

在这篇教程中，你可以学到如何创建 Storm 的拓扑（topology），并将他们发布到 Storm 集群中。本文主要使用 Java，不过也会少量使用 Python 来说明 Storm 的多语言支持能力。

## 序言

本教程通过 [storm-starter](https://github.com/apache/storm/blob/master/examples/storm-starter) 项目中的几个例子来介绍 Storm 的用法。强烈建议读者将该项目 clone 到本地并实际动手运行一下。你可以先按照[配置开发环境](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Setting-Up-A-Development-Environment.md)和[创建 Storm 项目](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Creating-A-New-Storm-Project.md)两篇文档的说明配置好你的开发环境。

## Storm 集群的组件

表面上看，Storm 的集群与 Hadoop 的集群很相似，只是在 Hadoop 集群上运行的是 “MapReduce jobs”，而 Storm 集群上运行的是拓扑（topologies）。实际上，Job 和 Topology 还是存在很大的差异的，其中一个重要的区别就是 MapReduce Job 是会主动运行结束的，而 Topology 却会持续不停地处理消息（除非你主动终止它）。

在 Storm 的集群中有两个类型的节点：master 节点和 worker 节点。在 master 节点上运行一个名为 “Nimbus” 的后台程序，这个程序有点类似于 Hadoop 的 “JobTracker”。Nimbus 负责管理整个集群的程序运行，将任务分配到集群的各个机器上，并负责监控任务执行是否成功。

每个 worker 节点都会运行一个名为 “Supervisor” 的后台进程。Supervisor 的主要功能是按照 Nimbus 的分配在它所在的机器上启动或停止 worker 进程，并监听机器上的所有相关任务的运行。一个 topology 由遍布于整个集群的数个 worker 进程组成，worker 进程所执行的就是 topology 的一个子集。

![storm-cluster](http://storm.apache.org/documentation/images/storm-cluster.png)

Nimbus 与各个 Supervisor 之间的通信都是通过 ZooKeeper 集群完成的。另外，Numbus 进程和 Supervisor 进程都是快速失败的、无状态的；它们的状态实际上是保存在 ZooKeeper 或者本地磁盘上的。也就是说，如果你使用 `kill -9` 命令终止了 Nimbus 和 Supervisor 进程，它们在重新启动之后会恢复之前的状态。这一点确保了 Storm 集群具有极高的稳定性。

## 拓扑

在 Storm 上运行实时计算任务首先就需要创建一个拓扑（topology）。拓扑是一个表征计算流的“图”。拓扑上的每个节点都对应一个处理逻辑，节点之间的连线表示了数据是如何在节点之间进行传递的。

拓扑的运行很简单。首先，将所有的代码和依赖包编译打包到一个单独的 jar 文件中，然后，在 Storm 集群上运行以下命令：

```
storm jar all-my-code.jar backtype.storm.MyTopology arg1 arg2
```

这个命令表示使用参数 `arg1` 和 `arg2` 来运行类 `backtype.storm.MyTopology`，该类中的 main 函数定义了拓扑的结构。执行该命令就会将拓扑提交到 Nimbus 上。该命令的 `storm jar` 部分就是用于和 Nimbus 建立连接并上传 jar 文件的。

因为拓扑是 Thrift 结构的，Nimbus 也是一种 Thrift 服务，因此理论上说可以使用任意一种语言来提交拓扑。上面的例子就是一个使用 JVM 语言的简单方法。关于如何启动/停止拓扑的更多详细信息，请参考[在生产环境中运行拓扑](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Running-Topologies-On-A-Production-Cluster.md)一文。

## 流

Storm 的核心概念是“流”（stream）。流是若干 tuple 组成的序列。Storm 提供了一种将一个流传输到另外一个流中的原语，这个传输过程是分布式的，而且非常可靠。例如，将一个微博消息的流传输到一个话题趋势的流中就是这样一个过程。

Spout 和 Bolt 是 Storm 中用于传输流的基本原语。通过 Spout 和 Bolt 的相关接口就可以实现应用程序的相关逻辑。

Spout 是流的来源。Spout 可以从一个 [Kestrel](http://github.com/nathanmarz/storm-kestrel) 消息队列中读取 tuple，并将这些 tuple 发射出去组成一个流。Spout 也可以通过微博网站提供的 API 读取微博消息并发射一个微博消息流。

Bolt 负责接收输入流中的消息数据，并对这些数据进行相应的处理，还可以根据需要发射出新的流。对于类似于从一个微博消息流中计算出话题趋势的消息流这样复杂的流传输过程，往往需要很多步骤并且需要很多个 Bolt。在 Bolt 中可以进行函数运行、tuple 过滤处理、流的联结与聚合、与数据库交互等操作。

Topology 可以看作一个由若干个 Spout 和 Bolt 组成的网络打包成的顶级抽象结构，可以把它直接提交到 Storm 集群中运行。Topology 实际上是一个流传输的图，其中的节点就是 Spout 或者 Bolt。图的边缘表示 Bolt 会订阅到哪个流。在 Spout 或者 Bolt 发射一个 tuple 到流中的时候，这个 tuple 最终会被发送到订阅该流的所有 Bolt 中。

![topology](http://storm.apache.org/documentation/images/topology.png)

拓扑图中的连线显示了 tuple 是如何在拓扑的各个节点中传递的。例如，如果从 Spout A 到 Bolt B 之间、从 Spout A 到 Bolt C 之间、从 Bolt B 到 Bolt C 之间各有一个有向连线，那么 Spout A 每次发射的 tuple 都会被发送到 Bolt B 和 Bolt C，同样，Bolt B 输出的所有 tuple 也会发送到 Bolt C。

拓扑中的每个节点是并行运行的。在拓扑中可以指定每个节点的并发度，Storm 会在整个集群中分配指定数量的线程来执行节点的任务。

如果不主动终止，Storm 中的拓扑会一直运行下去。Storm 还会自动重新分派执行失败的任务，确保即使在机器故障、消息丢失的场景下也不会发生数据丢失的情况。

## 数据模型
