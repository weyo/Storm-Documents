# 本地模式

本地模式是一种在本地进程中模拟 Storm 集群的工作模式，对于开发和测试拓扑很有帮助。在本地模式下运行拓扑与[在集群模式下][1]运行拓扑的方式很相似。

创建一个进程内的“集群”只需要使用 `LocalCluster` 类即可，例如：

```java
import backtype.storm.LocalCluster;

LocalCluster cluster = new LocalCluster();
```

随后，你就可以使用 `LocalCluster` 中的 `submitTopology` 方法来提交拓扑了。与 [StormSubmitter][2] 中相应的方法相似，`submitTopology` 接收一个拓扑名称、拓扑配置以及拓扑对象作为输入参数。你也可以以拓扑名称为参数，使用 `killTopology` 方法来 kill 掉对应的拓扑。

使用以下语句关闭本地模式集群运行：

```java
cluster.shutdown();
```

## 本地模式的常用配置

你可以在[这里][3]找到完整的配置项列表。以下是几个比较有用的配置项说明：

1. **Config.TOPOLOGY_MAX_TASK_PARALLELISM**：该配置项设置了单个组件（bolt/spout）的线程数上限。生产环境下的拓扑往往含有很高的并行度（数百个线程），导致在本地模式下测试拓扑时会有较大的负载。这个配置项可以让你很容易地控制并行度。
2. **Config.TOPOLOGY_DEBUG**：此配置项设置为 true 时 Storm 会打印出 spout 或者 bolt 每一次发送消息的日志记录。这个功能对于调试拓扑很有用。


[1]: http://storm.apache.org/documentation/Running-topologies-on-a-production-cluster.html
[2]: http://storm.apache.org/javadoc/apidocs/backtype/storm/StormSubmitter.html
[3]: http://storm.apache.org/javadoc/apidocs/backtype/storm/Config.html