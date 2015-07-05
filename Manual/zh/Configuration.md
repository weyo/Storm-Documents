# 配置

Storm 有大量配置项用于调整 nimbus、supervisors 和拓扑的行为。有些配置项是系统级的配置项，在拓扑中不能修改，另外一些配置项则是可以在拓扑中修改的。

每一个配置项都在 Storm 代码库的 [defaults.yaml][1] 中有一个默认值。可以通过在 Nimbus 和 Supervisors 的环境变量中定义一个 storm.yaml 来覆盖默认值。最后，在使用 [StormSubmitter][2] 提交拓扑时也可以定义基于具体拓扑的配置项。但是，基于拓扑的配置项仅仅能够覆盖那些以 “TOPOLOGY” 作为前缀的配置项。

Storm 0.7.0 以上版本支持覆写每个 Bolt/Spout 的配置信息。不过，使用这种方式只能修改以下几个配置项：

1. "topology.debug"
2. "topology.max.spout.pending"
3. "topology.max.task.parallelism"
4. "topology.kryo.register"：由于序列化对拓扑中的所有组件都是可见的，这一项与其他几项稍微有一些不同，详细信息可以参考 [Storm 的序列化][3]

Storm 的 Java API 支持两种自定义组件配置信息的方式：

1. 内置型：在需要配置的 Spout/Bolt 中覆写 `getComponentConfiguration` 方法，使其返回特定组件的配置表；
2. 外置型：`TopologyBuilder` 中的 `setSpout` 与 `setBolt` 方法会返回一个带有 `addConfiguration` 方法的 `ComponentConfigurationDeclarer` 对象，通过 `addConfiguration` 方法就可以覆写对应组件的配置项（同时也可以添加自定义的配置信息——译者注）。

配置信息的优先级依次为：defaults.yaml < storm.yaml < 拓扑配置 < 内置型组件信息配置 < 外置型组件信息配置。

**相关资料**

- [Config][4]：此类包含所有可配置项的列表，对于创建拓扑配置信息很有帮助
- [defaults.yaml][1]：所有配置项的默认值
- [配置 Storm 集群][5]：说明了如何创建、配置一个 Storm 集群
- [在生产环境中运行拓扑][6]：列出了在集群中运行拓扑的一些有用的配置项
- [本地模式][7]：列出了使用本地模式时比较有用的配置项


[1]: https://github.com/apache/storm/blob/master/conf/defaults.yaml
[2]: http://storm.apache.org/javadoc/apidocs/backtype/storm/StormSubmitter.html
[3]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Serialization.md
[4]: http://storm.apache.org/javadoc/apidocs/backtype/storm/Config.html
[5]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Setting-Up-A-Storm-Cluster.md
[6]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Running-Topologies-On-A-Production-Cluster.md
[7]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Local-Mode.md
