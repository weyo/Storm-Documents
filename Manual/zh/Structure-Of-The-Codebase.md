# 源码组织结构

Strom 的代码有三个层次：

第一，Storm 在一开始就是按照兼容多语言的目的来设计的。Nimbus 是一个 Thrift 服务，拓扑也被定义为 Thrift 架构。Thrift 的使用使得 Storm 可以用于任何一种语言。

第二，所有的 Storm 接口都设计为 Java 接口。所以，尽管 Storm 核心代码中有大量的 Clojure 实现，所有的访问都必须经过 Java API。这就意味着 Storm 的每个特性都可以通过 Java 来实现。

第三，Storm 的实现中大量使用了 Clojure。可以说，Storm 的代码结构大概是一半的 Java 代码加上一半的 Clojure 代码。但是由于 Clojure 更具有表现力，所以实际上 Storm 的核心逻辑大多是采用 Clojure 来实现的。

下面详细说明了每个层次的细节信息。

## storm.thrift

要理解 Storm 的代码架构，首先需要了解 [storm.thrift][1] 文件。

Storm 使用这个 [fork][2] 版本的 Thrift（“storm” 分支）来生成代码。这个 “fork” 版本实际上就是 Thrift7，其中所有的 Java package 也都重命名成了 `org.apache.thrift7`。在其他方面，它与 Thrift7 完全相同。这个 fork 主要是为了解决 Thrift 缺乏向后兼容的机制的问题，同时，也可以让用户在自己的 Storm 拓扑中使用其他版本的 Thrift。

拓扑中的每个 spout 或者 bolt 都有一个特定的标识，这个标识称为“组件 id”。组件 id 主要为了从拓扑中 spout 和 bolt 的输出流中选择一个或多个流作为某个 bolt 订阅的输入流。[Storm 拓扑][3]中就包含有一个组件 id 与每种类型的组件（spout 与 bolt）相关联的 map。

Spout 和 Bolt 有相同的 Thrift 定义。我们来看看 [Bolt 的 Thrift 定义][4]。它包含一个 `ComponentObject` 结构和一个 `ComponentCommon` 结构。

`ComponentObject` 定义了 bolt 的实现，这个实现可以是以下三种类型中的一种：

1. 一个 Java 序列化对象（实现了 [IBolt][5] 接口的对象）。
2. 一个用于表明其他语言的实现的 `ShellComponent` 对象。以这种方式指定一个 bolt 会让 Storm 实例化一个 [ShellBolt][6] 对象来处理基于 JVM 的 worker 进程与组件的非 JVM 实现之间的通信。
3. 一个带有类名与构造器参数的 Java 对象结构，Storm 可以使用这个结构来实例化 bolt。如果你需要定义一个非 JVM 语言的拓扑这个类型会很有用。使用这种方式，你可以在不创建并且序列化一个 Java 对象的情况下使用基于 JVM 的 spout 与 bolt。

`ComponentCommon` 定义了组件的其他方面特性，包括：

1. 该组件的输出流以及每个流的 metadata（无论是一个直接流还是基于域定义的流）；
2. 该组件消费的输入流（使用流分组所定义的一个将组件 id 与流 id 相关联的 map 来指定）；
3. 该组件的并行度；
4. 该组件的组件级配置。

注意，spout 的结构也有一个 `ComponentCommon` 域，所以理论上说 spout 也可以声明一个输入流。然而 Storm 的 Java API 并没有为 spout 提供消费其他的流的方法，并且如果你为 spout 声明了输入流，在提交拓扑的时候也会报错。这是因为 spout 的输入流声明不是为了用户的使用，而是为了 Storm 内部的使用。Storm 会为拓扑添加隐含的流与 bolt 来设置应答框架（acking framework）。这些隐含的流中就有两个流用于从 acker bolt 向拓扑中的每个 spout 发送消息。在发现 tuple 树完成或者失败之后，acker 就会通过这些隐含的流发送 “ack” 或者 “fail” 消息。将用户的拓扑转化为运行时拓扑的代码在[这里][7]。

## Java 接口

Storm 的对外接口基本上为 Java 接口，主要的几个接口有：

1. [IRichBolt][8]
2. [IRichSpout][9]
3. [TopologyBuilder][10]

大部分接口的策略为：

1. 使用一个 Java 接口来定义接口；
2. 实现一个具有适当的默认实现的 Base 类。

你可以从 [BaseRichSpout][11] 类中观察到这种策略的工作机制。

如上所述，Spout 和 Bolt 都已经根据拓扑的 Thrift 定义进行了序列化。

在这些接口中，`IBolt`、`ISpout` 与 `IRichBolt`、`IRichSpout` 之间存在着一些细微的差别。其中最主要的区别是带有 “Rich” 的接口中增加了 `declareOutputFields` 方法。这种区别的原因主要在于每个输出流的输出域声明必须是 Thrift 结构的一部分（这样才能实现跨语言操作），而用户本身只需要将流声明为自己的类的一部分即可。`TopologyBuilder` 在构造 Thrift 结构时所做的就是调用 `declareOutputFields` 方法来获取声明并将其转化为 Thrift 结构。这种转化过程可以在 `TopologyBuilder` 的[源码][12]中看到。

## 实现

通过 Java 接口来详细说明所有的功能可以确保 Storm 的每个特征都是有效的。更重要的是，关注 Java 接口可以让有 Java 使用经验的用户更易上手。

另一方面，Storm 的核心架构主要是通过 Clojure 实现的。尽管按照一般的计数规则来说代码库中 Java 与 Clojure 各占 50%，但是大部分逻辑实现还是基于 Clojure 的。不过也有两个例外，分别是 DRPC 和事务型拓扑的实现。这两个部分是完全使用 Java 实现的。这是为了说明在 Storm 中如何实现高级抽象。DRPC 和事务型拓扑的实现分别位于 [backtype.storm.coordination][13]、[backtype.storm.drpc][14] 和 [backtype.storm.transactional][15] 包中。

以下是主要的 Java 包和 Clojure 命名空间的总结。

### Java packages

[backtype.storm.coordination][16]: 实现了用于将批处理整合到 Storm 上层的功能，DRPC 和事务型拓扑都需要这个功能。`CoordinatedBolt` 是其中最重要的类。

[backtype.storm.drpc][14]: DRPC 高级抽象的实现。

[backtype.storm.generated][17]: 为 Storm 生成的 Thrift 代码（使用了这个 [fork][2] 版本的 Thrift，其中仅仅将包名重命名为 org.apache.thrift7 来避免与其他 Thrift 版本的冲突）。

[backtype.storm.grouping][18]: 包含自定义流分组的接口。

[backtype.storm.hooks][19]: 用于在 Storm 中添加事件钩子的接口，这些事件包括任务发送 tuple、tuple 被 ack 等等。

[backtype.storm.serialization][20]: Storm 序列化/反序列化 tuple 的接口。这是在 [Kryo][21] 的基础上构建的。

[backtype.storm.spout][22]: Spout 与一些关联接口的定义（例如 `SpoutOutputCollector`）。其中也包含有用于实现非 JVM 语言 spout 的协议的 `ShellSpout`。

[backtype.storm.task][23]: Bolt 与关联接口的定义（例如 `OutputCollector`）。其中也包含有用于实现非 JVM 语言 bolt 的协议的 `ShellBolt`。最后，`TopologyContext` 也是在这里定义的，该类可以用于在拓扑运行时为 spout 和 bolt 提供拓扑以及他们自身执行的相关信息。

[backtype.storm.testing][24]: 包含很多 bolt 测试类以及用于 Storm 单元测试的工具类。

[backtype.storm.topology][25]: 在 Thrift 结构上层的 Java 层，用于为 Storm 提供完全的 Java API（用户不必了解 Thrift）。`TopologyBuilder` 和一些为不同的 spout 和 bolt 提供帮助的基础类都在这里。稍微高级一点的 `IBasicBolt` 接口也在这里，该接口是一种实现基本的 bolt 的简单方式。

[backtype.storm.transactional][26]: 事务型拓扑的实现。

[backtype.storm.tuple][27]: Storm tuple 数据模型的实现。

[backtype.storm.utils][28]: 整个代码库中通用的数据结构和各种工具类。

### Clojure namespaces

[backtype.storm.bootstrap](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/bootstrap.clj): Contains a helpful macro to import all the classes and namespaces that are used throughout the codebase.

[backtype.storm.clojure](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/clojure.clj): Implementation of the Clojure DSL for Storm.

[backtype.storm.cluster](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/cluster.clj): All Zookeeper logic used in Storm daemons is encapsulated in this file. This code manages how cluster state (like what tasks are running where, what spout/bolt each task runs as) is mapped to the Zookeeper "filesystem" API.

[backtype.storm.command.*](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/command): These namespaces implement various commands for the `storm` command line client. These implementations are very short.

[backtype.storm.config](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/config.clj): Implementation of config reading/parsing code for Clojure. Also has utility functions for determining what local path nimbus/supervisor/daemons should be using for various things. e.g. the `master-inbox` function will return the local path that Nimbus should use when jars are uploaded to it.

[backtype.storm.daemon.acker](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/acker.clj): Implementation of the "acker" bolt, which is a key part of how Storm guarantees data processing.

[backtype.storm.daemon.common](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/common.clj): Implementation of common functions used in Storm daemons, like getting the id for a topology based on the name, mapping a user's topology into the one that actually executes (with implicit acking streams and acker bolt added - see `system-topology!` function), and definitions for the various heartbeat and other structures persisted by Storm.

[backtype.storm.daemon.drpc](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/drpc.clj): Implementation of the DRPC server for use with DRPC topologies.

[backtype.storm.daemon.nimbus](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/nimbus.clj): Implementation of Nimbus.

[backtype.storm.daemon.supervisor](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/supervisor.clj): Implementation of Supervisor.

[backtype.storm.daemon.task](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/task.clj): Implementation of an individual task for a spout or bolt. Handles message routing, serialization, stats collection for the UI, as well as the spout-specific and bolt-specific execution implementations.

[backtype.storm.daemon.worker](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/worker.clj): Implementation of a worker process (which will contain many tasks within). Implements message transferring and task launching.

[backtype.storm.event](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/event.clj): Implements a simple asynchronous function executor. Used in various places in Nimbus and Supervisor to make functions execute in serial to avoid any race conditions.

[backtype.storm.log](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/log.clj): Defines the functions used to log messages to log4j.

[backtype.storm.messaging.*](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/messaging): Defines a higher level interface to implementing point to point messaging. In local mode Storm uses in-memory Java queues to do this; on a cluster, it uses ZeroMQ. The generic interface is defined in protocol.clj.

[backtype.storm.stats](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/stats.clj): Implementation of stats rollup routines used when sending stats to ZK for use by the UI. Does things like windowed and rolling aggregations at multiple granularities.

[backtype.storm.testing](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/testing.clj): Implementation of facilities used to test Storm topologies. Includes time simulation, `complete-topology` for running a fixed set of tuples through a topology and capturing the output, tracker topologies for having fine grained control over detecting when a cluster is "idle", and other utilities.

[backtype.storm.thrift](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/thrift.clj): Clojure wrappers around the generated Thrift API to make working with Thrift structures more pleasant.

[backtype.storm.timer](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/timer.clj): Implementation of a background timer to execute functions in the future or on a recurring interval. Storm couldn't use the [Timer](http://docs.oracle.com/javase/1.4.2/docs/api/java/util/Timer.html) class because it needed integration with time simulation in order to be able to unit test Nimbus and the Supervisor.

[backtype.storm.ui.*](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/ui): Implementation of Storm UI. Completely independent from rest of code base and uses the Nimbus Thrift API to get data.

[backtype.storm.util](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/util.clj): Contains generic utility functions used throughout the code base.
 
[backtype.storm.zookeeper](https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/zookeeper.clj): Clojure wrapper around the Zookeeper API and implements some "high-level" stuff like "mkdirs" and "delete-recursive".


[1]: https://github.com/apache/storm/blob/master/storm-core/src/storm.thrift
[2]: https://github.com/nathanmarz/thrift/tree/storm
[3]: https://github.com/apache/storm/blob/master/storm-core/src/storm.thrift#L91
[4]: https://github.com/apache/storm/blob/master/storm-core/src/storm.thrift#L79
[5]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/task/IBolt.java
[6]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/task/ShellBolt.java
[7]: https://github.com/apache/storm/blob/master/storm-core/src/clj/backtype/storm/daemon/common.clj#L279
[8]: http://storm.apache.org/javadoc/apidocs/backtype/storm/topology/IRichBolt.html
[9]: http://storm.apache.org/javadoc/apidocs/backtype/storm/topology/IRichSpout.html
[10]: http://storm.apache.org/javadoc/apidocs/backtype/storm/topology/TopologyBuilder.html
[11]: http://storm.apache.org/javadoc/apidocs/backtype/storm/topology/base/BaseRichSpout.html
[12]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/topology/TopologyBuilder.java#L205
[13]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/coordination
[14]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/drpc
[15]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/transactional
[16]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/coordination
[17]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/generated
[18]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/grouping
[19]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/hooks
[20]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/serialization
[21]: http://code.google.com/p/kryo/
[22]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/spout
[23]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/task
[24]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/testing
[25]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/topology
[26]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/transactional
[27]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/tuple
[28]: https://github.com/apache/storm/tree/master/storm-core/src/jvm/backtype/storm/tuple
