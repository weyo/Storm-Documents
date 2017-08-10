# 分布式 RPC

分布式 RPC（DRPC）的设计目标是充分利用 Storm 的计算能力实现高密度的并行实时计算。Storm 接收若干个函数参数作为输入流，然后通过 DRPC 输出这些函数调用的结果。严格来说，DRPC 并不能算作是 Storm 的一个特性，因为它只是一种基于 Storm 原语 (Stream、Spout、Bolt、Topology) 实现的计算模式。虽然可以将 DRPC 从 Storm 中打包出来作为一个独立的库，但是与 Storm 集成在一起显然更有用。

## 概述

DRPC 是通过一个 DRPC 服务端(DRPC server)来实现分布式 RPC 功能的。DRPC server 负责接收 RPC 请求，并将该请求发送到 Storm 中运行的 Topology，等待接收 Topology 发送的处理结果，并将该结果返回给发送请求的客户端。因此，从客户端的角度来说，DPRC 与普通的 RPC 调用并没有什么区别。例如，以下是一个使用参数 “http://twitter.com” 调用 “reach” 函数计算结果的例子：

```java
DRPCClient client = new DRPCClient("drpc-host", 3772);
String result = client.execute("reach", "http://twitter.com");
```

下图是 DRPC 的原理示意图。

![DRPC][1]

客户端通过向 DRPC 服务器发送待执行函数的名称以及该函数的参数来获取处理结果。实现该函数的拓扑使用一个 `DRPCSpout` 从 DRPC 服务器中接收一个函数调用流。DRPC 服务器会为每个函数调用都标记了一个唯一的 id。随后拓扑会执行函数来计算结果，并在拓扑的最后使用一个名为 `ReturnResults` 的 bolt 连接到 DRPC 服务器，根据函数调用的 id 来将函数调用的结果返回。

## 定义 DRPC 拓扑

可以直接使用普通的拓扑构造方法来构造 DRPC 拓扑，如下所示：

```java
public static class ExclaimBolt extends BaseBasicBolt {
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        String input = tuple.getString(1);
        collector.emit(new Values(tuple.getValue(0), input + "!"));
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("id", "result"));
    }
}
public static void main(String[] args) throws Exception {
	TopologyBuilder builder = new TopologyBuilder();
	// builder.setSpout(drpcSpout);
	// builder.setBolt(new ExclaimBolt(), 3);
	// submit(builder.createTopology());
}
```

### 本地模式 DRPC

DRPC 可以在本地模式下运行。以下是使用本地模式构造拓扑的例子：

```java
LocalDRPC drpc = new LocalDRPC();
DRPCSpout spout = new DRPCSpout("exclamation", drpc);
builder.setSpout("drpc", spout);
builder.setBolt("exclaim", new ExclaimBolt(), 3)
		.shuffleGrouping("drpc");
builder.setBolt("return", new ReturnResults(), 3)
		.shuffleGrouping("exclaim");

LocalCluster cluster = new LocalCluster();
Config conf = new Config();
cluster.submitTopology("drpc-demo", conf, builder.createTopology());

// local mode 测试代码
System.out.println(drpc.execute("exclamation", "hello"));

cluster.shutdown();
drpc.shutdown();
```

在这种模式下，首先你会创建一个 `LocalDPRC` 对象，该对象会在进程中模拟一个 DRPC 服务器，其作用类似于 `LocalCluster` 在进程中模拟 Storm 集群的功能。在定义好拓扑的各个组件之后，就可以使用 `LocalCluster` 来提交拓扑。在本地模式下 `LocalDPRC` 对象不会绑定到任何一个实际的端口，所以需要通过向 `DRPCSpout` 传入参数的方式来关联到拓扑中。

在启动拓扑后，你可以使用 `execute` 方法来完成 DRPC 调用。

### 远程模式 DRPC

在一个实际的集群中使用 DRPC 有以下三个步骤：

1. 配置并启动 DRPC 服务器；
2. 在集群的各个服务器上配置 DRPC 服务器的地址；
3. 将 DRPC 拓扑提交到集群运行。

可以像 Nimbus、Supervisor 那样使用 `storm` 命令来启动 DRPC 服务器（注意，此 server 的基本配置，如 nimbus，ZooKeeper 等参数应该与 Storm 集群其他机器相同）：

```
bin/storm drpc
```

接下来，你需要在集群的各个服务器上配置 DRPC 服务器的地址。这是为了让 `DRPCSpout` 了解从哪里获取函数调用的方法。可以通过编辑 `storm.yaml` 或者添加拓扑配置的方式实现配置。配置 `storm.yaml` 的方式类似于下面这样：

```
drpc.servers:
  - "drpc1.foo.com"
  - "drpc2.foo.com"
```

最后，你可以像其他拓扑一样使用 `StormSubmitter` 来启动拓扑。

以下是使用远程模式构造拓扑的一个例子：

```java
TopologyBuilder builder = new TopologyBuilder();

DRPCSpout spout = new DRPCSpout("exclamation");
builder.setSpout("drpc", spout, 3);
builder.setBolt("exclaim", new ExclamationBolt(), 3)
        .shuffleGrouping("drpc");
builder.setBolt("return", new ReturnResults(), 7)
        .shuffleGrouping("exclaim");

Config conf = new Config();
conf.setNumWorkers(2);

StormSubmitter.submitTopology("drpc-demo", conf, builder.createTopology());
```

## 更复杂的例子

请参考[Trident 教程][2]一文中计算指定 URL 的 Reach 数的例子。


[1]: http://storm.apache.org/releases/0.9.6/images/drpc-workflow.png
[2]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Trident-Tutorial.md#reach