# Trident Spouts

与一般的 Storm API 一样，spout 也是 Trident 拓扑的数据来源。不过，为了实现更复杂的功能服务，Trident Spout 在普通的 Storm Spout 之上另外提供了一些 API 接口。

数据源、数据流以及基于数据流更新 state（比如数据库）的操作，他们之间的耦合关系是不可避免的。[Trident State][1] 一文中有这方面的详细解释，理解他们之间的这种联系对于理解 spout 的运作方式非常重要。

Trident 拓扑中的大部分 spout 都是非事务型 spout。在 Trident 拓扑中可以使用普通的 `IRichSpout` 接口来创建数据流：

```
TridentTopology topology = new TridentTopology();
topology.newStream("myspoutid", new MyRichSpout());
```

Trident 拓扑中的所有 spout 都必须有一个唯一的标识，而且这个标识必须在整个 Storm 集群中都是唯一的。Trident 需要使用这个标识来存储 spout 从 ZooKeeper 中消费的元数据（metadata），包括 txid 以及其他相关的 spout 元数据。

你可以使用以下配置项来设置用于存储 spout 元数据的 ZooKeeper 地址（一般情况下不需要设置以下选项，因为 Storm 默认会直接使用集群的 ZooKeeper 服务器来存储数据 —— 译者注）：

1. `transactional.zookeeper.servers`：ZooKeeper 的服务器列表
2. `transactional.zookeeper.port`：ZooKeeper 集群的端口
3. `transactional.zookeeper.root`：元数据在 ZooKeeper 中存储的根目录。元数据会直接存储在该设置目录下。

## 管道

默认情况下，Trident 每次处理只一个 batch，知道该 batch 处理成功或者失败之后才会开始处理其他的 batch。你可以通过将 batch 管道化来提高吞吐率，降低每个 batch 的处理延时。同时处理的 batch 的最大数量可以通过 `topology.max.spout.pending` 来进行配置。

不过，即使在同时处理多个 batch 的情况下，Trident 也会按照 batch 的顺序来更新 state。例如，假如你正在处理一个将全局计数结果聚合并更新到数据库中的任务，那么在你向数据库中更新 batch1 的计数结果时，你同时可以继续处理 batch2、batch3 甚至 batch10 的计数工作。不过，Trident 只会在 batch1 的 state 更新结束之后才会处理后续 batch 的 state 更新操作。这是实现恰好一次处理的语义的必要基础，我们已经在 [Trident State][1] 一文中讨论了这一点。

## Trident spout 类型

下面列出了一些可用的 spout API 接口：

1. [ITridentSpout][2]：这是最常用的 API，支持事务型和模糊事务型的语义实现。不过一般会根据需要使用它的某个已有的实现，而不是直接实现该接口。
2. [IBatchSpout][3]：非事务型 spout，每次会输出一个 batch 的 tuple。
3. [IPartitionedTridentSpout][4]：可以从分布式数据源（比如一个集群或者 Kafka 服务器）读取数据的事务型 spout。
4. [OpaquePartitionedTridentSpout][5]：可以从分布式数据源读取数据的模糊事务型 spout。

当然，正如这篇教程的开头提到的，除了这些 API 之外，你还可以使用普通的 `IRichSpout`。


[1]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Trident-State.md
[2]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/spout/ITridentSpout.java
[3]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/spout/IBatchSpout.java
[4]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/spout/IPartitionedTridentSpout.java
[5]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/spout/IOpaquePartitionedTridentSpout.java