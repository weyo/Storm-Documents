# 在生产环境中运行拓扑

在生产环境集群中运行拓扑的方式与[本地模式][1]非常相似，主要包括以下几个步骤：

1) 定义拓扑（如果使用 Java 进行开发就可以使用 [TopologyBuilder][2]）

2) 使用 [StormSubmitter][3] 向集群提交拓扑。`StormSubmitter` 接收拓扑名称、拓扑配置信息以及拓扑对象本身作为参数，如下所示：

```java
Config conf = new Config();
conf.setNumWorkers(20);
conf.setMaxSpoutPending(5000);
StormSubmitter.submitTopology("mytopology", conf, topology);
```

3) 将你的拓扑程序以及相关依赖库（除了 Storm 本身的依赖 —— 这些依赖已经添加到 Storm 的工作节点的 classpath 中了）打包为一个 jar 文件。

如果你使用 Maven 进行开发，可以使用 [Maven Assembly Plugin][4] 来打包，你需要做的仅仅是将下述插件配置添加到你的 pom.xml 中：

```xml
  <plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
      <descriptorRefs>  
        <descriptorRef>jar-with-dependencies</descriptorRef>
      </descriptorRefs>
      <archive>
        <manifest>
          <mainClass>com.path.to.main.Class</mainClass>
        </manifest>
      </archive>
    </configuration>
  </plugin>
```

然后就可以运行 `mvn assembly:assembly` 来打包。请确保你已经在 dependencies 中排除了 Storm 本身的 jar 包。

4) 使用 `storm` 客户端向集群提交拓扑，在提交时需要指定好你的 jar 包的相关路径、主函数所在类名称以及其他一些需要的参数，下面是一个提交拓扑的例子：

```
storm jar path/to/allmycode.jar org.me.MyTopology arg1 arg2 arg3
```

`storm jar` 会将 jar 提交到集群中，同时配置 `StormSubmitter` 类来与正确的集群建立连接。在上面的例子里，上传 jar 包之后，`storm jar` 就会使用 “arg1”、“arg2”、“arg3” 三个参数来运行 `org.me.MyTopology` 的 main 函数。

关于如何配置 Storm 客户端与 Storm 集群的交互的详细信息，请参阅[配置开发环境](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Setting-Up-A-Development-Environment.md)一文。

## 常用配置

拓扑中有很多参数可以设置。你可以在[这里][5]找到完整的配置项列表。其中，以 “TOPOLOGY” 开头的参数可以被拓扑中的对应配置项覆盖（其他参数是集群的配置参数，不能被直接覆盖）。以下是拓扑中的一些常用参数：

1. **Config.TOPOLOGY_WORKERS**：此项设置了可以用于执行拓扑的 worker 进程数。例如，如果你将该参数值设置为 25，那么在集群中就会有 25 个可以执行任务的 Java 进程。另外，如果你将拓扑的并行度设置成了 150，那么每个 worker 进程就会执行 6 个任务线程。
2. **Config.TOPOLOGY_ACKERS**：此项设置了用于跟踪 spout 发送的 tuple 树的 ack 任务数。Ackers 是 Storm 可靠性模型的重要组成部分，你可以在[消息的可靠性保障][6]一文中了解更多相信信息。
3. **Config.TOPOLOGY_MAX_SPOUT_PENDING**：此项设置了单个 Spout 任务能够挂起的最大的 tuple 数（tuple 挂起表示该 tuple 已经被发送但是尚未被 ack 或者 fail）。强烈建议设置此参数来防止消息队列的爆发性增长。
4. **Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS**：此项设置了 ackers 跟踪 tuple 的超时时间。默认值是 30 秒，对于大部分拓扑而言这个值基本上是不需要改动的。关于 Storm 的消息可靠性模型请参考[消息的可靠性保障][6]一文。
5. **Config.TOPOLOGY_SERIALIZATIONS**：此项用于在 Storm 中注册更多的序列化工具，这样你就可以使用自定义的序列化类型来处理 tuple。

## Kill 拓扑

执行以下命令来 kill 拓扑：

`storm kill {topologyname}`

其中 `topologyname` 就是你提交拓扑时使用的拓扑名称。

不过，在执行该命令后 Storm 不会马上 kill 掉该拓扑。Storm 会先停止所有 spouts 的活动，使得他们不能继续发送 tuple，然后 Storm 会等待 `Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS` 参数表示的一段时间，然后才会结束所有的 worker 进程。这可以保证拓扑在被 kill 之前可以有足够的时间完成已有的 tuple 的处理。

## 更新运行中的拓扑

目前只能通过先 kill 掉当前的拓扑再重新提交新拓扑的方式来更新运行中的拓扑。不过社区计划在将来实现一个 `storm swap` 命令来将一个运行中的拓扑替换为一个新的拓扑，尽可能减少停机时间，同时确保不会有两个拓扑同时处理 tuple 的情况发生。

## 监控拓扑

监控拓扑运行的最好方式是使用 Storm UI。Storm UI 可以显示任务中的错误信息以及每个运行中拓扑中每个组件的吞吐量与端到端延时的性能信息。

当然，你也可以通过查看在工作节点机器上的日志信息来了解拓扑运行情况。


[1]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Local-Mode.md
[2]: http://storm.apache.org/javadoc/apidocs/backtype/storm/topology/TopologyBuilder.html
[3]: http://storm.apache.org/javadoc/apidocs/backtype/storm/StormSubmitter.html
[4]: http://maven.apache.org/plugins/maven-assembly-plugin/
[5]: http://storm.apache.org/javadoc/apidocs/backtype/storm/Config.html
[6]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Guaranteeing-Message-Processing.md