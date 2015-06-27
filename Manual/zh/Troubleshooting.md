# 问题与解决

本文介绍了用户在使用 Storm 过程中遇到的问题与相应的解决方法。

## Worker 进程在启动时挂掉而没有留下堆栈跟踪信息的问题

可能出现的现象：

- 拓扑在一个节点上运行正常，但是多个 worker 进程在多个节点上就会崩溃

解决方案：

- 你的网络配置可能有问题，导致每个节点无法根据 hostname 连接到其他的节点。ZeroMQ 有时会在不能识别 host 的时候挂掉 进程。如果是这种情况，有两种可行的解决方案：
	- 在 /etc/hosts 文件中配置好 hostname 与 IP 的对应关系
	- 设置一个局域网 DNS 服务器，使得节点可以根据 hostname 定位到其他节点

## 节点之间无法通信

可能出现的现象：

- 每个 spout tuple 的处理都不成功
- 拓扑中的处理过程不起作用

解决方案：

- Storm 不支持 ipv6，你可以在 supervisor 的 child-opts 配置中添加 `-Djava.net.preferIPv4Stack=true` 参数，然后重启 supervisor。
- 你的网络配置可能存在问题，请参考上个问题中的解决方案。

## 拓扑在一段时间后停止了 tuple 的处理过程

可能出现的现象：

- 拓扑正常运行一段时间后突然停止了数据处理过程，并且 spout 的 tuple 一起开始处理失败

解决方案：

- 这是 ZeroMQ 2.1.10 中的一个已经确认的问题，请将 ZMQ 降级到 2.1.7 版本。

## Storm UI 中没有显示出所有的 supervisor 信息

可能出现的现象：

- Storm UI 中缺少部分 supervisor 的信息
- 在刷新 Storm UI 页面后 supervisor 列表会变化

解决方案：

- 确保 supervisor 的本地工作目录是相互独立的（也就是说不要出现在 NFS 中共享同一个目录的情况）
- 尝试删除 supervisor 的本地工作目录，然后重启 supervisor 后台进程。supervisor 启动时会为自己创建一个唯一的 id 并存储在本地目录中。如果这个 id 被复制到其他节点中，就会让 Storm 无法确定哪个 supervisor 正在运行（这种情况并不少见，如果需要扩展集群，就很容易出现直接将某个节点的 Storm 文件直接复制到新节点的情况 —— 译者注）。

## “Multiple defaults.yaml found” 错误

可能出现的现象：

- 在使用 `storm jar` 命令部署拓扑时出现此错误

解决方案：

- 你很可能在拓扑的 jar 包中包含了 Storm 自身的 jar 包。注意，在打包拓扑时，请不要将 Storm 自身的 jar 包加入，因为 Storm 已经在它的 classpath 中提供了这些 jar 包。

## 运行 storm jar 命令时出现 “NoSuchMethorError”

可能出现的现象：

- 运行 `storm jar` 命令时出现奇怪的 “NoSuchMethodError”

解决方案：

- 这可能是由于你部署拓扑的 Storm 版本与你构建拓扑时使用的 Storm 版本不同。请确保你编译拓扑时使用的 Storm 版本与你运行拓扑的 Storm 客户端版本相同。

## Kryo ConcurrentModificationException

可能出现的现象：

- 系统运行时出现如下的异常堆栈跟踪信息

```
java.lang.RuntimeException: java.util.ConcurrentModificationException
    at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:84)
    at backtype.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:55)
    at backtype.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:56)
    at backtype.storm.disruptor$consume_loop_STAR_$fn__1597.invoke(disruptor.clj:67)
    at backtype.storm.util$async_loop$fn__465.invoke(util.clj:377)
    at clojure.lang.AFn.run(AFn.java:24)
    at java.lang.Thread.run(Thread.java:679)
Caused by: java.util.ConcurrentModificationException
    at java.util.LinkedHashMap$LinkedHashIterator.nextEntry(LinkedHashMap.java:390)
    at java.util.LinkedHashMap$EntryIterator.next(LinkedHashMap.java:409)
    at java.util.LinkedHashMap$EntryIterator.next(LinkedHashMap.java:408)
    at java.util.HashMap.writeObject(HashMap.java:1016)
    at sun.reflect.GeneratedMethodAccessor17.invoke(Unknown Source)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:616)
    at java.io.ObjectStreamClass.invokeWriteObject(ObjectStreamClass.java:959)
    at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1480)
    at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1416)
    at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1174)
    at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:346)
    at backtype.storm.serialization.SerializableSerializer.write(SerializableSerializer.java:21)
    at com.esotericsoftware.kryo.Kryo.writeClassAndObject(Kryo.java:554)
    at com.esotericsoftware.kryo.serializers.CollectionSerializer.write(CollectionSerializer.java:77)
    at com.esotericsoftware.kryo.serializers.CollectionSerializer.write(CollectionSerializer.java:18)
    at com.esotericsoftware.kryo.Kryo.writeObject(Kryo.java:472)
    at backtype.storm.serialization.KryoValuesSerializer.serializeInto(KryoValuesSerializer.java:27)
```

解决方案：

- 这个信息表示你在将一个可变的对象作为 tuple 发送出去。你发送到 outputcollector 中的所有对象必须是非可变的。这个错误表明对象在被序列化并发送到网络中时你的 bolt 正在修改这个对象。

## Storm 中的 NullPointerException

可能出现的现象：

- Storm 运行中出现了如下的 NullPointerException

```
java.lang.RuntimeException: java.lang.NullPointerException
    at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:84)
    at backtype.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:55)
    at backtype.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:56)
    at backtype.storm.disruptor$consume_loop_STAR_$fn__1596.invoke(disruptor.clj:67)
    at backtype.storm.util$async_loop$fn__465.invoke(util.clj:377)
    at clojure.lang.AFn.run(AFn.java:24)
    at java.lang.Thread.run(Thread.java:662)
Caused by: java.lang.NullPointerException
    at backtype.storm.serialization.KryoTupleSerializer.serialize(KryoTupleSerializer.java:24)
    at backtype.storm.daemon.worker$mk_transfer_fn$fn__4126$fn__4130.invoke(worker.clj:99)
    at backtype.storm.util$fast_list_map.invoke(util.clj:771)
    at backtype.storm.daemon.worker$mk_transfer_fn$fn__4126.invoke(worker.clj:99)
    at backtype.storm.daemon.executor$start_batch_transfer__GT_worker_handler_BANG_$fn__3904.invoke(executor.clj:205)
    at backtype.storm.disruptor$clojure_handler$reify__1584.onEvent(disruptor.clj:43)
    at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:81)
    ... 6 more
```

解决方案：

- 这个问题是由于多个线程同时调用 `OutputCollector` 中的方法造成的。Storm 中所有的 emit、ack、fail 方法必须在同一个线程中运行。出现这个问题的一种场景是在一个 `IBasicBolt` 中创建了一个独立的线程。由于 `IBasicBolt` 会在 `execute` 方法调用之后自动调用 `ack`，所以这就会出现多个线程同时使用 `OutputCollector` 的情况，进而抛出这个异常。也就是说，在使用 `IBasicBolt` 时，所有的消息发送操作必须在同一个线程的 `execute` 方法中执行。
