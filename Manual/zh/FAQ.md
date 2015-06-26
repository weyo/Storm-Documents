# FAQ

## Storm 最佳实践

### 关于配置 Storm + Trident 的建议

- worker 的数量最好是服务器数量的倍数；topology 的总并发度(parallelism)最好是 worker 数量的倍数；Kafka 的分区数(partitions)最好是 Spout（特指 ` KafkaSpout `）并发度的倍数
- 在每个机器（supervisor）上每个拓扑应用只配置一个 worker
- 在拓扑最开始运行的时候设置使用较少的大聚合器，并且最好是每个 worker 进程分配一个
- 使用独立的调度器（scheduler）来分配任务（关于Scheduler 的知识请参考 [xumingming 的博客][1] —— 译者注）
- 在每个 worker 上只配置使用一个 acker —— 这是 0.9.x 版本的默认特性，不过在早期版本中有所不同
- 在配置文件中开启 GC 日志记录；如果一切正常，日志中记录的 major GC 应该会非常少
- 将 trident 的 batch interval 配置为你的集群的端到端时延的 50% 左右
- 开始时设置一个很小的 `TOPOLOGY_MAX_SPOUT_PENDING`（对于 trident 可以设置为 1，对于一般的 topology 可以设置为 executor 的数量），然后逐渐增大，直到数据流不再发生变化。这时你可能会发现结果大约等于 `“2 × 吞吐率(每秒收到的消息数) × 端到端时延”` （最小的额定容量的2倍）。

### 如何避免 worker 总是出现莫名其妙的故障的问题

- 确保 Storm 对你的日志目录有写权限
- 确保你的堆内存没有溢出
- 确保所有的 worker 上都已经正确地安装了所有的依赖库文件
- 确保 ZooKeeper 的 hostname 不是简单地设置为 “localhost”
- 确保集群中的每台机器上都配置好了正确、唯一的 hostname，并且这些 hostname 需要配置在所有机器的 Storm 配置文件中
- 确保 a) 不同的 worker 之间，b) 不同的 Storm 节点之间，c) Storm 与 ZooKeeper 集群之间， d) 各个 worker 与拓扑运行所需要的 Kafka/Kestrel/其他数据库等 之间没有开启防火墙或者其他安全保护机制；如果有，请使用 netstat 来为各个端口之间的通信授权

### Help！Storm 使用过程中无法获取：

- **日志文件**：日志文件默认记录在 `$STORM_HOME/logs` 目录中。请检查你对该目录是否有写权限。具体的日志配置信息位于 logback/cluster.xml 文件中（0.9 之前的版本需要在 log4j/*.properties 配置文件中进行配置。
- **最终输出的 JVM 设置**：需要在配置文件（storm.yaml）的 `childopts` 配置项中添加 `-XX+PrintFlagsFinal` 命令选项。
- **最终输出的 Java 系统属性信息**：需要在你构建拓扑的位置添加代码 ```Properties props = System.getProperties(); props.list(System.out);```

### 我应该使用多少个 worker？

worker 的完整数量是由 supervisor 配置的。每个 supervisor 会分配到一定数量的 JVM slot，你在拓扑中设置的 worker number 就是以这个 slot 数量为依据进行分配的。

不建议为每个拓扑在每台机器上分配超过一个 worker。

假如有一个运行于三台 8 核服务器节点的拓扑，它的并行度为24，每个 bolt 在每台机器上分配有 8 个 executor（即每个 CPU 核心分配一个）。这种场景下，使用三个 worker （每个 worker 分配 8 个executor）相对于使用更多的 worker （比如使用 24 个 worker，为每个 executor 分别分配一个）有三点好处：

首先，在 worker 内部将数据流重新分区到不同的 executor 的操作（比如 shuffle 或者 group-by）就不会产生触发到传输 buffer 缓冲区，tuple 会直接从发送端转储到接收端的 buffer 区。这一点有很大的优势。相反，如果目标 executor 是在同一台机器的不同 worker 进程内，tuple 就需要经历“发送 -> worker 传输队列 -> 本地 socket 端口 -> 接收端 worker -> 接收端 executor”这样一个漫长的过程。虽然这个过程并不会产生网络级传输，但是在同一台机器的不同进程间的传输损耗也是很可观的。

其次，三个大的聚合器带来的大的缓存空间比 24 个小聚合器带来的小缓存空间要有用得多。因为这回降低数据倾斜造成的影响，同时提高 LRU 的性能。

最后，更少的 worker 可以有效地降低控制流的频繁变动。

## 拓扑

### Trident 拓扑支持多数据流吗

> Trident 拓扑可以设计成条件路径（if-else）的工作流形式吗？例如，bolt0 在接收 spout 的数据流时，可以根据输入 tuple 的值来选择将数据流发送到 bolt1 或者 bolt2，而不是同时向两个 bolt 发送。

Trident 的 “each” 运算符可以返回一个数据流对象，你可以将该对象存储在某个变量中，然后你可以对同一个数据流执行多个 each 操作来分解该数据流，如下述代码所示：

```java
Stream s = topology.each(...).groupBy(...).aggregate(...) 
Stream branch1 = s.each(..., FilterA) 
Stream branch2 = s.each(..., FilterB) 
```

你可以使用 join、merge 或者 multiReduce 来联结各个数据流。

到目前为止，Trident 暂时不支持输出多个数据流。（详见 [STORM-68][2]）

## Spout

### Coordinator 是什么，为什么会有很多 Coordinator？

Trident spout 实际上是通过 Storm 的 bolt 运行的。`MasterBatchCoordinator`（MBC）封装了 Trident 拓扑的 spout，它负责整合 Trident 中的 batch，这一点对于你所使用的任何类型的 spout 而言都是一样的。Trident 的 batch 就是在 MBC 向各个 spout-coordinator 分发种子 tuple 的过程中生成的。Spout-coordinator bolt 知道你所定义的 spout 是如何互相协作的 —— 实际上，在使用 Kafka 的情况下，各个 spout 就是通过 spout-coordinator 来获取 pull 消息所需要的 partition 和 offset 信息的。

### 在 spout 的 metadata 记录中能够存储什么信息？

只能存储少量静态数据，而且是越少越好（尽管你确实可以向其中存储更多的信息，不过我们不推荐这样做）。

### `emitPartitionBatchNew` 函数是多久调用一次的？

由于在 Trident 中 MBC 才是实际运行的 spout，一个 batch 中的所有 tuple 都是 MBC 生成的 tuple 树的节点。也就是说，Storm 的 “max spout pending” 参数实际上定义的是可以并发运行的 batch 数量。MBC 在满足以下两个条件下会发送出一个新的 batch：首先，挂起的 tuple 数需要小于 “max pending” 参数；其次，距离上一个 batch 的发送已经过去了至少一个 [trident batch interval][3] 的间隔时间。

### 如果没有数据发送，Trident 会降低发送频率吗？

是的，Storm 中有一个可选的 “spout 等待策略”，默认配置是 sleep 一段指定的[配置时间][4]。

### Trident batch interval 参数有什么用？

你知道 486 时代的计算机上面为什么有个 [trubo button][5] 吗？这个参数的作用和这个按钮有点像。

实际上，trident batch interval 有两个用处。首先，它可以用于减缓 spout 从远程数据源获取数据的速度，但这不会影响数据处理的效率。例如，对于一个从给定的 S3 存储区中读取批量上传文件并按行发送数据的 spout，我们就不希望它经常触发 S3 的阈值，因为文件要隔几分钟才会上传一次，而且每个 batch 也需要花费一定的时间来执行。

另一个用处是限制启动期间或者突发数据负载情况下内部消息队列的负载压力。如果 spout 突然活跃起来，并向系统中挤入了 10 个 batch 的记录，那么可能会有从 batch7 开始的大量不紧急的 tuple 堵塞住传输缓冲区，并且阻塞了从 batch3 中的 tuple（甚至可能包含 batch3 中的部分旧 tuple）的 commit 过程<sup>*</sup>。对于这种情况，我们的解决方法就是将 trident batch interval 设置为正常的端到端处理时延的一半左右 —— 也就是说如果需要花费 600 ms 的时间处理一个 batch，那么就可以每 300 ms 处理一个 batch。

注意，这个 300 ms 仅仅是一个上限值，而不是额外增加的延时时间，如果你的 batch 需要花费 258 ms 来运行，那么 Trident 就只会延时等待 42 ms。

### 如何设置 batch 大小？

Trident 本身不会对 batch 进行限制。不过如果使用 Kafka 的相关 spout，那么就可以使用 max fetch bytes 大小除以 平均 record 大小来计算每个子 batch 分区的有效 record 大小。

### 怎样重新设置 batch 的大小？

Trident 的 batch 在某种意义上是一种过载的设施。batch 大小与 partition 的数量均受限于或者是可以用于定义<sup>*</sup>：

1. 事务安全单元（一段时间内存在风险的 tuple）；
2. 相对于每个 partition，一个用于窗口数据流分析的有效窗口机制；
3. 相对于每个 partition，使用 partitionQuery，partitionPersist 等命令时能够同时进行的查询操作数量；
4. 相对于每个 partition，spout 能够同时分配的 record 数量。

不能在 batch 生成之后更改 batch 的大小，不过可以通过 shuffle 操作以及修改并行度的方式来改变 partition 的数量。

## 时间相关问题

### 怎样基于指定时间聚合数据

对于带有固定时间戳的 records，如果需要对他们执行计数、求均值或者聚合操作，并将结果整合到离散的时间桶（time bucket）中，Trident 是一个很好的具有可扩展性的解决方案。

这种情况下可以写一个 `each` 函数来将时间戳置入一个时间桶中：如果桶的大小是以“小时”为单位的，那么时间戳 `2013-08-08 12:34:56` 就会被匹配到 `2013-08-08 12:00:00` 桶中，其他的 12 时到 13 时之间的时间也一样。然后可以使用 `persistentAggregate` 来对时间桶分组。`persistentAggregate` 会使用一个基于数据存储的本地 cacheMap。这些包含有大量 records 的 group 会使用高效的批量读取/写入方式对数据存储区进行操作，所以并不会对数据存储区进行大量的读操作；只要你的数据传送足够快捷，Trident 就可以高效地使用内存与网络。即使某台服务器宕机了一天，需要重新快速地发送一整天的数据，旧有的结果也可以静默地获取到并进行更新，并且这并不会影响当前结果的计算过程。

### 怎么才能知道某个时间桶中已经收到了所有需要的 record？

很遗憾，你不会知道什么时候所有的 event 都已经采集到了 —— 这是一个认识论问题，而不是一个分布式系统的问题。你可以：

- 使用域相关知识来设定时间限制。
- 引入标记机制：对于一个指定时间窗，确定某个 record 会处在所有的 record 的最后位置。Trident 使用这个机制来判断一个 batch 是否结束。例如，你收到一组传感器采集到的 records，每个传感器都是按顺序发送数据的，那么一旦所有的传感器都发送出一个 “3:02:xx” 的数据，你就可以知道可以开始处理这个时间窗了。
- 如果可以的话，尽量使你的处理过程增量化：每个新来的值都会使结果越来越准确。Trident ReducerAggregator 就是一个可以通过一个旧有的结果以及一组新数据来返回一个更新的结果的运算符。这使得结果可以被缓存并序列化到一个数据库中；如果某台服务器宕机了一天，在恢复运行之后需要重新快速地发送一整天的数据，旧有的结果也可以静默地获取到并进行更新。
- 使用 Lambda 架构：将所有收到的事件数据归档到存储区中（S3，HBase，HDFS）。在快速处理层，一旦时间窗复位，就对对应的时间桶进行处理来获取有效结果，并且在处理过程中跳过所有比早于该时间窗的过期数据。定期地执行全局聚合操作就可以计算出一个较“正确”的结果。


>## _附注_
<sup>*</sup> 此处译文可能不够准确，有需要的读者请参考原文


[1]: https://xumingming.sinaapp.com/854/twitter-storm-pluggable-scheduler/
[2]: https://issues.apache.org/jira/browse/STORM-68
[3]: https://github.com/apache/storm/blob/master/conf/defaults.yaml#L115
[4]: https://github.com/apache/storm/blob/master/conf/defaults.yaml#L110
[5]: http://en.wikipedia.org/wiki/Turbo_button
